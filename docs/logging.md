# Logging

Flask, FastAPI, gunicorn, and even docker have some form of logging, or integrate the python logging library within them. This page specfically discusses about just the python default logging library.

## Basic Log

Below describes a simple logging assignment with basic config.

```python
import logging

logging.basicConfig(
    filename='model.log', filemode = 'a',
    level=logging.DEBUG,
    format='%(asctime)s.%(msecs)03d %(levelname)s %(module)s - %(funcName)s: %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S')

logging.debug("A Debug Logging Message")
logging.info("A Info Logging Message")
logging.warning("A Warning Logging Message")
logging.error("An Error Logging Message")
logging.critical("A Critical Logging Message")
```

Logging comes with different levels, indicating their severity. This is what can be seen from the log file using the above example.

```log
2021-10-11 14:32:33.445 DEBUG logging_test - main: A Debug Logging Message
22021-10-11 14:32:33.445 INFO logging_test - main: A Info Logging Message
32021-10-11 14:32:33.445 WARNING logging_test - main: A Warning Logging Message
42021-10-11 14:32:33.445 ERROR logging_test - main: An Error Logging Message
52021-10-11 14:32:33.445 CRITICAL logging_test - main: A Critical Logging Message
```

## Rotating Log

The limitation of the basic log configuration is that it cannot set a hard limit on the size of logs being collected, and can overload the server's harddisk over time.

We can use the `RotatingFileHandler` in this case.

```python
import logging
from logging.handlers import RotatingFileHandler


rfh = logging.handlers.RotatingFileHandler(
    filename='response.log', #path & name of logfile
    mode='a',
    maxBytes=100*1024*1024, #100mb limit for each file
    backupCount=3, #max of 4 files to be stored, therefore 400mb of logs will be saved
    encoding=None,
    delay=0
)

logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s %(message)s",
    datefmt="%y-%m-%d %H:%M:%S",
    handlers=[rfh]
)

logger = logging.getLogger('main')
```

We can see that multiple log files with `response.log*` will be saved, each with a maximum of 100Mb.

```bash
.
├── [  18]  requirements.txt
├── [ 95M]  response.log
├── [100M]  response.log.1
├── [100M]  response.log.2
└── [1.0K]  rotatinglogger.py
```


### Other handlers

There are many other [handlers]((https://docs.python.org/3/library/logging.handlers.html)) for specific use-cases.