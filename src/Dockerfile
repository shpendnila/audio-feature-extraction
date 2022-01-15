FROM amazon/aws-lambda-python:3.8

COPY app.py ./

COPY bin ./

RUN yum -y install qt
RUN yum -y install qt5-qtbase
RUN yum -y install qt5-qtbase-common

ENV VAMP_PATH $pwd"vamp"

CMD ["app.handler"]


