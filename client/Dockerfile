FROM jupyterhub/k8s-binderhub:0.2.0-n361.h6f57706

RUN mkdir /usr/local/share/ca-certificates/extra
COPY nginx/jovyan.example.com.crt /usr/local/share/ca-certificates/extra/
RUN update-ca-certificates
