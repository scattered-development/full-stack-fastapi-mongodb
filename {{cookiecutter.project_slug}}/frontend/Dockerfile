FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /frontend

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else npm install && npm ci; \
  fi

# Development image with hot reload (used only by docker-compose.override.yml)
FROM deps AS dev
WORKDIR /frontend
EXPOSE 80
# Re-sync node_modules against the bind-mounted package-lock.json on every
# container start, rather than trusting the named volume's seeded-once content
# (Docker only copies the image's node_modules into the volume the first time
# it's created — rebuilding the image later never refreshes an existing volume).
# Plain `npm install` (not `npm ci`) so this is a fast no-op when nothing
# changed and only does real work when the lockfile did.
CMD npm install && npm run dev -- -p 80 -H 0.0.0.0

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /frontend
COPY --from=deps /frontend/node_modules ./node_modules
COPY . .

# Next.js collects completely anonymous telemetry data about general usage.
# Learn more here: https://nextjs.org/telemetry
# Uncomment the following line in case you want to disable telemetry during the build.
ENV NEXT_TELEMETRY_DISABLED 1

# NEXT_PUBLIC_* vars are baked into the client bundle at build time, so this must be
# set here (not just in .env) to override the localhost default in .env.local for
# non-local builds.
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}

# If using npm comment out above and use below instead
RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /frontend

ENV NODE_ENV production
# Uncomment the following line in case you want to disable telemetry during runtime.
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /frontend/public ./public

# Set the correct permission for prerender cache
RUN mkdir .next
RUN chown nextjs:nodejs .next

# Automatically leverage output traces to reduce image size
# https://nextjs.org/docs/advanced-features/output-file-tracing
COPY --from=builder --chown=nextjs:nodejs /frontend/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /frontend/.next/static ./.next/static

USER nextjs

EXPOSE 80
ENV PORT 80

# set hostname to localhost
ENV HOSTNAME "0.0.0.0"

CMD [ "node", "server.js" ]