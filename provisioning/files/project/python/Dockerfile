FROM python:3.12.3-alpine3.18
ENV APP_ROOT /src
ENV CONFIG_ROOT /config
RUN mkdir ${CONFIG_ROOT}
COPY requirements.txt ${CONFIG_ROOT}/requirements.txt
RUN pip install -r ${CONFIG_ROOT}/requirements.txt
RUN mkdir ${APP_ROOT}
RUN django-admin startproject mysite ${APP_ROOT}
WORKDIR ${APP_ROOT}

