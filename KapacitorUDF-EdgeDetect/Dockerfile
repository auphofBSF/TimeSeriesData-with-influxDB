FROM python:2.7-alpine
ADD . /code
ADD . /kapacitor
WORKDIR /code

# UNCOMMENT following if we have any dependency requirements
#RUN pip install -r requirements.txt


#As no PyPi install image exists
#REF: Publish Python UDF package on PyPi. #930 - https://github.com/influxdata/kapacitor/issues/930

#Image can be copied from
#https://dl.influxdata.com/kapacitor/releases/python-kapacitor_udf-1.4.0.tar.gz
#TODO: How do we get this into 
#Build the Kapacitor (Python) Agent, copied from GitHub


#The following expects to have the code somehow in docker for the udf agent
#RUN python setup.py install

#The following expects to have the CURL in alpine that is not the case
#RUN  curl -fsSL https://dl.influxdata.com/kapacitor/releases/python-kapacitor_udf-1.4.0.tar.gz -o /tmp/python_kapacitor_udf-1.4.0.tar.gz \

RUN python -c "import urllib;testfile = urllib.URLopener();testfile.retrieve('https://dl.influxdata.com/kapacitor/releases/python-kapacitor_udf-1.4.0.tar.gz', '/tmp/python_kapacitor_udf-1.4.0.tar.gz')" \
  \
  && cd /tmp \
  && echo "056d0968daa4fdb12373f3d62d568863ef1a5338  /tmp/python_kapacitor_udf-1.4.0.tar.gz" | sha1sum -c - \
  && tar -xzf /tmp/python_kapacitor_udf-1.4.0.tar.gz  --directory /code \
  \
  && cd /code/kapacitor_udf-1.4.0 \
  \
  && python setup.py install \
  \
  && rm -rf /tmp/python_kapacitor* 

WORKDIR /code/udf

#CMD ["python", "mirror.py"]
CMD ["python", "edgedetect.py"]