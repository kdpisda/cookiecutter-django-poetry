{% if cookiecutter.frontend_pipeline in ['Gulp', 'Webpack'] -%}
FROM node:16-bullseye-slim as client-builder

ARG APP_HOME=/app
WORKDIR ${APP_HOME}

COPY ./package.json ${APP_HOME}
RUN npm install && npm cache clean --force
COPY . ${APP_HOME}
{%- if cookiecutter.frontend_pipeline == 'Webpack' and cookiecutter.use_whitenoise == 'n' %}
{%- if cookiecutter.cloud_provider == 'AWS' %}
ARG DJANGO_AWS_STORAGE_BUCKET_NAME
ENV DJANGO_AWS_STORAGE_BUCKET_NAME=${DJANGO_AWS_STORAGE_BUCKET_NAME}
ARG DJANGO_AWS_S3_CUSTOM_DOMAIN
ENV DJANGO_AWS_S3_CUSTOM_DOMAIN=${DJANGO_AWS_S3_CUSTOM_DOMAIN}
{%- elif cookiecutter.cloud_provider == 'GCP' %}
ARG DJANGO_GCP_STORAGE_BUCKET_NAME
ENV DJANGO_GCP_STORAGE_BUCKET_NAME=${DJANGO_GCP_STORAGE_BUCKET_NAME}
{%- elif cookiecutter.cloud_provider == 'Azure' %}
ARG DJANGO_AZURE_ACCOUNT_NAME
ENV DJANGO_AZURE_ACCOUNT_NAME=${DJANGO_AZURE_ACCOUNT_NAME}
{%- endif %}
{%- endif %}
RUN npm run build

{%- endif %}
# define an alias for the specific python version used in this file.
FROM python:3.11.3-slim-bullseye as python

# Python build stage
FROM python as python-build-stage

ARG BUILD_ENVIRONMENT=production

# Install apt packages
RUN apt-get update && apt-get install --no-install-recommends -y \
  # dependencies for building Python packages
  build-essential \
  # psycopg2 dependencies
  libpq-dev \
  # Install Poetry
  curl \
  && curl -sSL https://install.python-poetry.org | POETRY_HOME=/opt/poetry bash \
  && ln -s /opt/poetry/bin/poetry /usr/local/bin/poetry

# Copy pyproject.toml and poetry.lock files
COPY pyproject.toml poetry.lock ./

# Set the virtualenv creation to false and install dependencies using Poetry
RUN poetry config virtualenvs.create false \
  && poetry install --no-root --no-dev

# Python 'run' stage
FROM python as python-run-stage

ARG BUILD_ENVIRONMENT=production
ARG APP_HOME=/app

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1
ENV BUILD_ENV ${BUILD_ENVIRONMENT}

WORKDIR ${APP_HOME}

RUN addgroup --system django \
    && adduser --system --ingroup django django

# Install required system dependencies
RUN apt-get update && apt-get install --no-install-recommends -y \
  # psycopg2 dependencies
  libpq-dev \
  # Translations dependencies
  gettext \
  # cleaning up unused files
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && rm -rf /var/lib/apt/lists/*

# Copy the installed dependencies from the build stage
COPY --from=python-build-stage /opt/poetry /opt/poetry
RUN ln -s /opt/poetry/bin/poetry /usr/local/bin/poetry


COPY --chown=django:django ./compose/production/django/entrypoint /entrypoint
RUN sed -i 's/\r$//g' /entrypoint
RUN chmod +x /entrypoint


COPY --chown=django:django ./compose/production/django/start /start
RUN sed -i 's/\r$//g' /start
RUN chmod +x /start


{%- if cookiecutter.use_celery == "y" %}
COPY --chown=django:django ./compose/production/django/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker
RUN chmod +x /start-celeryworker


COPY --chown=django:django ./compose/production/django/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat
RUN chmod +x /start-celerybeat


COPY ./compose/production/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower
RUN chmod +x /start-flower
{%- endif %}


# copy application code to WORKDIR
{%- if cookiecutter.frontend_pipeline in ['Gulp', 'Webpack'] %}
COPY --from=client-builder --chown=django:django ${APP_HOME} ${APP_HOME}
{% else %}
COPY --chown=django:django . ${APP_HOME}
{%- endif %}

# make django owner of the WORKDIR directory as well.
RUN chown django:django ${APP_HOME}

USER django

ENTRYPOINT ["/entrypoint"]
