# Integración de Next.js + NextAuth + Prisma + Stripe (suscripciones)

Este documento organiza y corrige el código que compartiste para que funcione en **App Router** de Next.js.

## 1) Variables de entorno (`.env`)

```bash
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DB"

NEXTAUTH_SECRET="clave_super_segura"
NEXTAUTH_URL="https://tudominio.com"

STRIPE_SECRET_KEY="sk_live_xxx"
STRIPE_WEBHOOK_SECRET="whsec_xxx"
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_live_xxx"
```

---

## 2) Prisma (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String         @id @default(uuid())
  email         String         @unique
  password      String?
  createdAt     DateTime       @default(now())
  subscriptions Subscription[]
}

model Subscription {
  id        String   @id @default(uuid())
  userId    String
  plan      String
  status    String
  stripeId  String?
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])
}
```

---

## 3) Cliente Stripe (`src/lib/stripe.ts`)

```ts
import Stripe from "stripe";

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: "2024-06-20",
});
```

---

## 4) Checkout API (`src/app/api/checkout/route.ts`)

```ts
import { stripe } from "@/lib/stripe";
import { NextResponse } from "next/server";

export async function POST(req: Request) {
  const { priceId, email } = await req.json();

  const session = await stripe.checkout.sessions.create({
    customer_email: email,
    mode: "subscription",
    line_items: [
      {
        price: priceId,
        quantity: 1,
      },
    ],
    success_url: `${process.env.NEXTAUTH_URL}/dashboard`,
    cancel_url: `${process.env.NEXTAUTH_URL}`,
  });

  return NextResponse.json({ url: session.url });
}
```

> Nota: `payment_method_types` es opcional en versiones recientes de Stripe Checkout para tarjeta.

---

## 5) Webhook Stripe (`src/app/api/webhooks/stripe/route.ts`)

```ts
import { stripe } from "@/lib/stripe";
import { NextResponse } from "next/server";
import prisma from "@/lib/prisma";

export async function POST(req: Request) {
  const sig = req.headers.get("stripe-signature");

  if (!sig) {
    return NextResponse.json({ error: "Missing Stripe signature" }, { status: 400 });
  }

  const body = await req.text();

  let event;

  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch {
    return NextResponse.json({ error: "Webhook error" }, { status: 400 });
  }

  if (event.type === "checkout.session.completed") {
    const session = event.data.object;

    // Recomendado: guardar userId en metadata al crear la sesión de checkout
    const userId = session.metadata?.userId;

    if (userId) {
      await prisma.subscription.create({
        data: {
          userId,
          plan: "Plan",
          status: "active",
          stripeId:
            typeof session.subscription === "string"
              ? session.subscription
              : session.subscription?.id,
        },
      });
    }
  }

  return NextResponse.json({ received: true });
}
```

Correcciones aplicadas frente al snippet original:
- Se eliminó `buffer` de `micro` (no se usa en App Router de esta forma).
- Se eliminó `export const config = { api: { bodyParser: false } }` (patrón de Pages Router).
- Se reemplazó el `userId` hardcodeado por `session.metadata.userId`.

---

## 6) NextAuth Credentials (`src/app/api/auth/[...nextauth]/route.ts`)

```ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import prisma from "@/lib/prisma";
import bcrypt from "bcrypt";

const handler = NextAuth({
  providers: [
    Credentials({
      name: "Credentials",
      credentials: {
        email: {},
        password: {},
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) return null;

        const user = await prisma.user.findUnique({
          where: { email: credentials.email },
        });

        if (!user?.password) return null;

        const valid = await bcrypt.compare(credentials.password, user.password);

        if (!valid) return null;

        return {
          id: user.id,
          email: user.email,
        };
      },
    }),
  ],
  secret: process.env.NEXTAUTH_SECRET,
});

export { handler as GET, handler as POST };
```

---

## 7) Dashboard (`src/app/dashboard/page.tsx`)

```tsx
"use client";

import { useSession } from "next-auth/react";

export default function Dashboard() {
  const { data } = useSession();

  return (
    <div>
      <h1>Panel de Usuario</h1>
      <p>Email: {data?.user?.email}</p>

      <div>
        <h2>Tu Plan</h2>
        <p>Activo</p>
      </div>
    </div>
  );
}
```

---

## 8) Función cliente para comprar (corregida)

```ts
const buy = async () => {
  const res = await fetch("/api/checkout", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      priceId: "price_xxx",
      email: "cliente@email.com",
    }),
  });

  const data = await res.json();
  window.location.href = data.url;
};
```

Corrección clave: había un typo en `da ta.url`.

---

## 9) Recomendaciones rápidas de producción

- No uses claves `sk_live` en local; usa `sk_test`.
- Verifica firma del webhook siempre (ya está contemplado).
- No guardes plan desde `display_items` (campo no confiable/legacy). Usa `priceId` y catálogo interno.
- Relaciona usuario ↔ checkout session vía `metadata.userId`.
- Añade idempotencia para webhooks (guardar `event.id` procesado).
