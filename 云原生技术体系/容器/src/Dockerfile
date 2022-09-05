FROM python:3

WORKDIR /usr/src/app

RUN pip3 install  --trusted-host pypi.python.org  Flask

COPY . .

EXPOSE 80

CMD [ "python3", "./helloworld.py" ]
~                                            
