FROM nginx/unit:1.25.0-python3.9
ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1

WORKDIR /webapp
COPY requirements.txt /webapp/
RUN pip3 install -r requirements.txt                                       \
    && apt autoremove --purge -y                                              \
    && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/*.list
COPY unit_config.json /docker-entrypoint.d/
COPY src /webapp/src
WORKDIR /webapp/
WORKDIR /webapp/src
RUN SECRET_KEY=empty python manage.py collectstatic --noinput
EXPOSE 80
VOLUME ["/media"]
