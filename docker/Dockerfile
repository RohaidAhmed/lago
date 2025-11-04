# ---------------------------------------------------------
# Stage 1: Build the frontend (Vite + PNPM)
# ---------------------------------------------------------
ARG NODE_VERSION=20
ARG RUBY_VERSION=3.4.7

FROM node:${NODE_VERSION}-alpine AS front_build

WORKDIR /app

COPY ./front/ .

RUN apk add --no-cache python3 build-base && \
    corepack enable && corepack prepare pnpm@latest --activate && \
    rm -rf node_modules && \
    pnpm install && pnpm build && pnpm prune --prod

# ---------------------------------------------------------
# Stage 2: Build Ruby dependencies (Rails API)
# ---------------------------------------------------------
FROM ruby:${RUBY_VERSION}-slim AS api_build

ENV BUNDLER_VERSION="2.5.5"
ENV PATH="$PATH:/root/.cargo/bin/"

WORKDIR /app

RUN apt-get update -y && apt-get install -y \
    build-essential libpq-dev nodejs git pkg-config libclang-dev libyaml-dev curl && \
    curl https://sh.rustup.rs -sSf | bash -s -- -y

COPY ./api/Gemfile ./api/Gemfile.lock ./
RUN gem install bundler --no-document -v "${BUNDLER_VERSION}" && \
    gem install foreman && \
    bundle config build.nokogiri --use-system-libraries && \
    bundle install --jobs=3 --retry=3 --without development test

# ---------------------------------------------------------
# Stage 3: Final runtime image
# ---------------------------------------------------------
FROM ruby:${RUBY_VERSION}-slim

WORKDIR /app

# Install Nginx and system dependencies
RUN apt-get update -y && apt-get install -y \
    nginx curl git libpq-dev ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Copy Nginx config
COPY ./docker/nginx.conf /etc/nginx/sites-enabled/default

# Copy built frontend and Ruby bundles
COPY --from=front_build /app/dist /app/front
COPY --from=api_build /usr/local/bundle /usr/local/bundle

# Copy Rails API and runner
COPY ./api ./api
COPY ./docker/Procfile ./api/Procfile
COPY ./docker/runner.sh ./runner.sh
COPY ./front/.env.sh ./front/.env.sh

RUN chmod +x ./runner.sh

ENV RAILS_ENV=production
ENV PORT=3000

EXPOSE 80
EXPOSE 3000

ENTRYPOINT ["./runner.sh"]
