{
  "name": "mh-selfcheck",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "@auth/prisma-adapter": "^1.0.0",
    "@prisma/client": "^5.12.0",
    "next": "13.5.6",
    "next-auth": "^4.24.5",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "stripe": "^12.0.0"
  },
  "devDependencies": {
    "prisma": "^5.12.0",
    "typescript": "^5.3.3"
  }
}
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id             String          @id @default(cuid())
  email          String          @unique
  name           String?
  image          String?
  emailVerified  DateTime?
  accounts       Account[]
  sessions       Session[]
  subscription   Subscription?
}

model Subscription {
  id              String   @id @default(cuid())
  stripeId        String   @unique
  status          String
  priceId         String
  currentPeriodEnd DateTime
  user            User     @relation(fields: [userId], references: [id])
  userId          String   @unique
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}
DATABASE_URL="postgresql://your-db-url"
GITHUB_ID=yourgithubclientid
GITHUB_SECRET=yourgithubsecret
STRIPE_SECRET_KEY=sk_test_xxx
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
NEXTAUTH_SECRET=super-secret-key
NEXTAUTH_URL=http://localhost:3000
NEXT_PUBLIC_BASE_URL=http://localhost:3000
import NextAuth from "next-auth";
import GitHubProvider from "next-auth/providers/github";
import { PrismaAdapter } from "@auth/prisma-adapter";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export const authOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
  ],
  callbacks: {
    async session({ session, user }) {
      const sub = await prisma.subscription.findUnique({
        where: { userId: user.id },
      });
      session.user.id = user.id;
      session.user.isPremium = sub?.status === "active";
      return session;
    },
  },
};

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
import { NextRequest, NextResponse } from "next/server";
import Stripe from "stripe";
import { getServerSession } from "next-auth";
import { authOptions } from "../auth/[...nextauth]/route";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2025-02-11.acacia",
});

export async function POST(req: NextRequest) {
  const session = await getServerSession(authOptions);
  if (!session?.user?.id) {
    return NextResponse.json({ error: "Not authenticated" }, { status: 401 });
  }

  const { priceId } = await req.json();

  const checkoutSession = await stripe.checkout.sessions.create({
    mode: "subscription",
    line_items: [{ price: priceId, quantity: 1 }],
    customer_email: session.user.email!,
    success_url: `${process.env.NEXT_PUBLIC_BASE_URL}/dashboard?success=true`,
    cancel_url: `${process.env.NEXT_PUBLIC_BASE_URL}/dashboard?canceled=true`,
  });

  return NextResponse.json({ url: checkoutSession.url });
}
import { NextRequest, NextResponse } from "next/server";
import Stripe from "stripe";
import { PrismaClient } from "@prisma/client";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2025-02-11.acacia",
});
const prisma = new PrismaClient();

export async function POST(req: NextRequest) {
  const sig = req.headers.get("stripe-signature")!;
  const body = await req.text();

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err: any) {
    return NextResponse.json({ error: err.message }, { status: 400 });
  }

  switch (event.type) {
    case "checkout.session.completed": {
      const session = event.data.object as Stripe.Checkout.Session;
      const customerEmail = session.customer_email!;
      const user = await prisma.user.findUnique({ where: { email: customerEmail } });
      if (user) {
        await prisma.subscription.upsert({
          where: { userId: user.id },
          update: {
            stripeId: session.subscription as string,
            status: "active",
            priceId: "default",
            currentPeriodEnd: new Date(),
          },
          create: {
            userId: user.id,
            stripeId: session.subscription as string,
            status: "active",
            priceId: "default",
            currentPeriodEnd: new Date(),
          },
        });
      }
      break;
    }
    case "customer.subscription.deleted": {
      const sub = event.data.object as Stripe.Subscription;
      await prisma.subscription.updateMany({
        where: { stripeId: sub.id },
        data: { status: "canceled" },
      });
      break;
    }
  }
  return NextResponse.json({ received: true });
}
export default function Home() {
  return (
    <main className="p-8">
      <h1 className="text-3xl font-bold">Mental Health Self-Check</h1>
      <p className="mt-2">Track your mood and upgrade to premium for extra features.</p>
    </main>
  );
}
"use client";
import { signIn } from "next-auth/react";

export default function LoginPage() {
  return (
    <main className="p-8">
      <h1 className="text-xl font-bold">Login</h1>
      <button
        onClick={() => signIn("github")}
        className="mt-4 px-4 py-2 rounded-xl bg-black text-white"
      >
        Sign in with GitHub
      </button>
    </main>
  );
}
"use client";
import { signOut } from "next-auth/react";

export default function LogoutPage() {
  return (
    <main className="p-8">
      <h1 className="text-xl font-bold">Logout</h1>
      <button
        onClick={() => signOut({ callbackUrl: "/" })}
        className="mt-4 px-4 py-2 rounded-xl bg-gray-700 text-white"
      >
        Log out
      </button>
    </main>
  );
}
import { getServerSession } from "next-auth";
import { authOptions } from "../api/auth/[...nextauth]/route";

export default async function Dashboard() {
  const session = await getServerSession(authOptions);

  if (!session) return <p>Please log in</p>;
  if (!session.user?.isPremium) return <p>You need premium access</p>;

  return <div className="p-8">✅ Premium Dashboard with mood tracking</div>;
}
