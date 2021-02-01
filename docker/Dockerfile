FROM python:3.7

RUN mkdir /my-application
WORKDIR /my-application
ADD . /my-application
RUN pip install -r requirements.txt

EXPOSE 5000
CMD ["python", "app.py"]
