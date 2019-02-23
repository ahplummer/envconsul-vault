# base image
FROM alpine:3.7

# set version
ENV ENVCONSUL_VERSION 0.7.3

# create a new directory
RUN mkdir /envconsul

# download dependencies
RUN apk --no-cache add \
      bash \
      ca-certificates \
      wget

# download and set up envconsul
RUN wget --quiet --output-document=/tmp/envconsul.zip https://releases.hashicorp.com/envconsul/${ENVCONSUL_VERSION}/envconsul_${ENVCONSUL_VERSION}_linux_amd64.zip && \
    unzip /tmp/envconsul.zip -d /envconsul && \
    rm -f /tmp/envconsul.zip && \
    chmod +x /envconsul

# update PATH
ENV PATH="PATH=$PATH:$PWD/envconsul"

# add the test runner
COPY ./testapp.sh /envconsul/testapp.sh
COPY ./config.json /envconsul/config.json
COPY ./config.hcl /envconsul/config.hcl

# expose port 8200
#EXPOSE 8200

# run vault
#ENTRYPOINT ["testapp.sh"]