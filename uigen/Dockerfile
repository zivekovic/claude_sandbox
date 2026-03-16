FROM node:20-alpine AS base

# Install dependencies needed for bcrypt native module
RUN apk add --no-cache libc6-compat python3 make g++

WORKDIR /app

# Copy package files
COPY package.json package-lock.json ./

# Install dependencies
RUN npm ci

# Copy the rest of the source
COPY . .

# Generate Prisma client
RUN npx prisma generate

# Run migrations and start dev server
EXPOSE 3000

CMD ["sh", "-c", "npx prisma migrate deploy && npm run dev"]
