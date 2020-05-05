FROM quay.io/jupyteronopenshift/s2i-minimal-notebook-py36

USER root

COPY ./source/anomaly-detection/. /opt/app-root/src/

RUN pip install boto3 && pip install boto
RUN pip install pyspark
RUN pip install pandas
RUN pip install sklearn


RUN yum install sudo -y

RUN sudo chmod -R 777 /opt/app-root/src

RUN sudo mkdir /opt/app-root/src/.local
RUN sudo chmod -R 777 /opt/app-root/src/.local

RUN sudo mkdir /opt/app-root/src/.local/share
RUN sudo chmod -R 777 /opt/app-root/src/.local/share

RUN sudo mkdir /opt/app-root/src/.local/share/jupyter
RUN sudo chmod -R 777 /opt/app-root/src/.local/share/jupyter


RUN sudo mkdir /opt/app-root/src/.local/share/jupyter/runtime
RUN sudo chmod -R 777 /opt/app-root/src/.local/share/jupyter/runtime

USER 0

RUN jupyter trust *.ipynb

RUN sudo chmod -R 777 /opt/app-root/src/.local/share/jupyter/notebook_secret

RUN sudo chmod -R 777 /opt/app-root/src/.pki/nssdb
RUN sudo chmod -R 777 /opt/app-root/src/.local/share/jupyter/nbsignatures.db

