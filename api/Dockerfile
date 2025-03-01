# syntax=docker/dockerfile:1

# Automatically build image using Python version specified in the `Pipfile`.
ARG API_PY_VERSION

##################
# Audio waveform #
##################

FROM realies/audiowaveform:latest AS awf

# Identify dependencies of the `audiowaveform` binary and move them to `/deps`,
# while retaining their folder structure
RUN ldd /usr/local/bin/audiowaveform | tr -s '[:blank:]' '\n' | grep '^/' | \
  xargs -I % sh -c 'mkdir -p $(dirname deps%); cp % deps%;'

##################
# Python builder #
##################

FROM python:${API_PY_VERSION} AS builder

# Container optimizations
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_NO_COLOR=1

# Activate the virtualenv
ENV PATH="/venv/bin:$PATH"

# - Install system packages needed for building Python dependencies
# - Create a virtualenv inside `/venv`
# - Install Pipenv to install Python dependencies
RUN apt-get update \
  && apt-get install -y python3-dev \
  && rm -rf /var/lib/apt/lists/* \
  && python -m venv /venv \
  && pip install --upgrade pipenv

# Copy the Pipenv files into the container
COPY Pipfile Pipfile.lock ./

# Install Python dependencies system-wide (uses the active virtualenv)
RUN pipenv install --system --deploy --dev

#######
# API #
#######

FROM python:${API_PY_VERSION}-slim AS api

LABEL org.opencontainers.image.source="https://github.com/WordPress/openverse"

# Container optimizations
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_NO_COLOR=1

# Activate the virtualenv
ENV PATH="/venv/bin:$PATH"

WORKDIR /api

ADD api/utils/fonts/SourceSansPro-Bold.ttf /usr/share/fonts/truetype/SourceSansPro-Bold.ttf

# Copy virtualenv from the builder image
COPY --from=builder /venv /venv

# Copy `audiowaveform` dependencies. This is unreliable as we use
# The `latest` version of the audiowaveform image which may introduce
# dependency changes which *could* have a different directory structure.
# If this step is failing, try adding the logging in this commit to help
# update the dependency paths:
# https://github.com/WordPress/openverse/commit/6cd8e3944a1d4ba7a3e80705b969a6a50eb75b5a
COPY --from=awf /deps/lib/ /lib/
COPY --from=awf /deps/usr/ /usr/

# Copy `audiowaveform` binary
COPY --from=awf /usr/local/bin/audiowaveform /usr/local/bin

# - Install system packages needed for running Python dependencies
#   - libexempi8: required for watermarking
#   - libpq-dev: required by `psycopg2`
# - Create directory for dumping API logs
RUN apt-get update \
  && apt-get install -y curl libpq-dev libexempi8 postgresql-client \
  && rm -rf /var/lib/apt/lists/* \
  && mkdir -p /var/log/openverse_api/openverse_api.log

# Create a folder for placing static files
RUN mkdir /static

# Create a non-root user, and make it the owner of the static dir created above
RUN useradd --create-home opener \
  && chown -R opener /static
USER opener

# Copy code into the final image
COPY --chown=opener . /api/

# Collect static assets, these are used by the next stage, `nginx`
RUN env \
  SETUP_ES="False" \
  DJANGO_SECRET_KEY="any string" \
  python manage.py collectstatic

# Add the release version to the docker container
ARG SEMANTIC_VERSION
ENV SEMANTIC_VERSION=$SEMANTIC_VERSION

# Exposes
# - 8000: Dev server for API Django app
EXPOSE 8000

# Wait for ES to accept connections
ENTRYPOINT ["./run.sh"]

# gunicorn configuration in gunicorn.conf.py
CMD ["gunicorn"]

#########
# NGINX #
#########

FROM nginx:1.25.2-alpine as nginx

LABEL org.opencontainers.image.source = "https://github.com/WordPress/openverse"

WORKDIR /app

COPY nginx.conf.template /etc/nginx/templates/openverse-api.conf.template

# Copy static files from `api` target
COPY --from=api /static /app/static

# Only environment variables with this prefix will be available in the template
ENV NGINX_ENVSUBST_FILTER="DJANGO_NGINX_"
ENV DJANGO_NGINX_ENVIRONMENT="local"
# Add the release version to the docker container
ARG SEMANTIC_VERSION
ENV DJANGO_NGINX_GIT_REVISION=$SEMANTIC_VERSION
