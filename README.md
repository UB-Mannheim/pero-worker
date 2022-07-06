# pero-worker

Project contains "worker" and "watchdog" for OCR processing system that uses pero-ocr package.

Processing system uses 5 components.
- Worker for processing data
- Watchdog for task planing and scheduling
- RabbitMQ message broker for task distribution
- Zookeeper for coordination of the workers and storing the configuration
- SFTP for storing OCR binary files.

## setup

Docker is used in this example. Please visit https://docs.docker.com/engine/install/ and follow instructions for your operating system.
Use installation instruction for Apache zookeeper, RabbitMQ and your favourite SFTP server, if you don't want to use docker.

Installation script was tested under Debian 11 and is APT dependent.

Installing requirements and create python venv for the project:
```
sh install_dependencies.sh
```

Source created virtual environment:
```
. ./.venv/bin/activate
```

Download pero-ocr to `pero-ocr/` folder:
```
git submodule init
git submodule update
```
Or do this manually by cloning pero-ocr from https://github.com/DCGM/pero-ocr.git

Starting required services.
Mounted paths are required for data persistency.
Service can recover this data after restart, so it does not have to be set up again from scratch.
```
docker run -d --rm -p2181:2181 -v /home/"$USER"/zookeeper-data:/data -v /home/"$USER"/zookeeper-datalog:/datalog --name="zookeeper" zookeeper
```
```
docker run -d --rm --name rabbitmq --hostname "$(hostname)" -v /home/"$USER"/rabbitmq:/var/lib/rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
```
```
 docker run --rm -d -p 2222:22 -v /home/"$USER"/ftp:/home/pero/ --name sftp atmoz/sftp:alpine pero:pero
```

## Initial system configuration

Set default server addresses and ports for auto-configuration:
```
python scripts/config_manager.py -z 127.0.0.1 -s 127.0.0.1 --ftp-servers 127.0.0.1:2222 --update-mq-servers --update-ftp-servers --update-monitoring-servers
```

Create processing stages for OCR pipeline:
```
python scripts/config_manager.py --name ocr_stage_x --config path/to/ocr_stage_x/config.ini --remote-path path/to/additional/data/on/ftp/server.tar.xz
```
Please note that you must upload additional files to SFTP server manually. Command above specifies just path used by worker to download these files from the server. To upload files use your favourite SFTP client.

For more details on configurations please visit pero-ocr git (https://github.com/DCGM/pero-ocr) and webpage (https://pero.fit.vutbr.cz/) to get more information.

Create output queue from where results can be downloaded. Output queue is stage without processing configuration.
```
python scripts/config_manager.py --name out
```

## Running worker and watchdog

```
python worker/worker.py -z 127.0.0.1
```
```
python worker/worker_watchdog.py -z 127.0.0.1
```

## Processing

Uploading images for processing:
```
python scripts/publisher.py --stages stage1 stage2 stage3 out --images input/file/1 input/file/2
```

Downloading results:
```
python scripts/publisher.py --directory output/directory/path --download out
```
If you want to keep downloading images from ```out``` stage, add ```--keep-running``` argument at the end of the command above.


## Additional info

System was tested with these versions of libraries:
```
kazoo==2.8.0
pika==1.2.0
protobuf==3.19.4
python-magic==0.4.25
requests==2.27.1
numpy==1.21.5
opencv-python==4.5.5.62
lxml==4.7.1
scipy==1.7.3
numba==0.55.1
torch==1.10.2
torchvision==0.11.3
brnolm==0.2.0
scikit-learn==1.0.2
scikit-image==0.19.1
tensorflow-gpu==2.8.0
shapely==1.8.0
pyamg==4.2.1
imgaug==0.4.0
arabic_reshaper==2.1.3
```
Python version used during development was `Python 3.9.2` but it should work with latest versions of python and libraries as well.