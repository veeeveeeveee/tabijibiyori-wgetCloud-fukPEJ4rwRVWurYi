
![](https://img2024.cnblogs.com/blog/1060878/202501/1060878-20250108173430136-1245556677.png)


本篇介绍使用Fastapi \+ sqlalchemy \+ alembic 来完成后端服务的数据库管理，并且通过docker\-compose来部署后端服务和数据库Mysql。包括：


1. 数据库创建，数据库用户创建
2. 数据库服务发现
3. Fastapi 连接数据库
4. Alembic 连接数据库
5. 服务健康检查


# 部署数据库



```
version: '3'
services:
  db:
    image: mysql
    container_name: db
    environment:
      - MYSQL_ROOT_PASSWORD=tv_2024 # root用户密码
      - MYSQL_DATABASE=tileView
      - MYSQL_USER=tile_viewer
      - MYSQL_PASSWORD=tv_2024
      - TZ=Asia/Shanghai
    volumes:
      - ./mysql:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 3306:3306
    restart: always

```

部署数据库有三个注意点：


1. 将数据库文件映射出来，避免丢失数据


数据库中存储的数据都容器里在/var/lib/mysql目录下，将该目录映射出来，避免重启容器丢失数据


二、自动创建数据库DB


很多情况下需要在启动数据库容器是自动创建数据库，在environment中设置 `MYSQL_DATABASE=tileView`即可在容器启动是创建数据库


三、创建可读写可远程用户


默认情况下非root用户不支持远程连接读写权限，在environment中设置


1. MYSQL\_USER\=tile\_viewer
2. MYSQL\_PASSWORD\=tv\_2024


将会获得一个可远程可读写MYSQL\_DATABASE库的用户


# Fastapi 连接数据库


非docker部署情况下使用IP和端口连接数据库，使用docker\-compose部署时服务都是自动化启动，事先不知道数据库的IP,这是就可以使用docker\-compose提供的能力：使用服务名来请求服务。


docker\-compose中可以使用服务的名称来通信，在通信请求中将服务名替换成容器IP。


首先将数据库连接的URL映射到服务的容器中



```
version: '3'
services:
  db:
    image: mysql
    container_name: db
    environment:
      - MYSQL_ROOT_PASSWORD=tv_2024 # root用户密码
      - MYSQL_DATABASE=tileView
      - MYSQL_USER=tile_viewer
      - MYSQL_PASSWORD=tv_2024
      - TZ=Asia/Shanghai
    volumes:
      - ./mysql:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 3306:3306
    restart: always
    healthcheck:
      test: [ "CMD", "mysqladmin", "ping", "-h", "localhost" ]
      interval: 10s
      timeout: 5s
      retries: 3
  server:
    image: tileview:1.0
    restart: always
    container_name: tileview_server
    ports:
      - "9100:9100"
    volumes:
      - ./tiles_store:/app/server/tiles_store
      - ./log:/app/server/log
      - ./upload:/app/server/upload
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DATABASE_URI=mysql+pymysql://tile_viewer:tv_2024@db/tileView

```

depends\_on: 表示该服务依赖db服务，db服务要先启动。`condition: service_healthy`


DATABASE\_URI：表示将数据库连接信息注入到该容器中，其中@db表示数据库IP:端口号 使用数据库服务名称db来替换，在容器中请求该url时，docker\-compose会自动将db转换成对应的数据库的IP和端口号。



```
depends_on:
      - db
    environment:
      - DATABASE_URI=mysql+pymysql://tile_viewer:tv_2024@db/tileView

```

# 修改sqlalchemy中数据库的连接


sqlalchemy 连接数据库时，数据库的url配置从环境变量中获取


sqlalchemy/database.py



```
import os

from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = os.getenv("DATABASE_URI")
engine = create_engine(SQLALCHEMY_DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

Base = declarative_base()


```

使用环境变量获取数据库



```
SQLALCHEMY_DATABASE_URL = os.getenv("DATABASE_URI")

```

在Fastapi 服务 调用数据库的方法。



```
DATABASE_URI=mysql+pymysql://tile_viewer:tv_2024@db/tileView

```

使用数据库服务名db来找到服务地址，在docker\-compose中会将服务名解析成服务的IP。在使用时会将db解析成172\.10\.0\.2，找到服务的IP。


# 迁移工具alembic中数据库连接


在数据库迁移工具中需要配置数据库的连接信息，该信息是配置在alembic.ini中，如果要使用环境变量中动态获取的方法，需要改变两点：


1. alembic.ini中不配置数据库连接信息
2. 在alembic/env.py动态获取连接url，并设置到alembic.ini中


alembic.ini中不配置数据库连接信息


![](https://cdn.nlark.com/yuque/0/2024/png/12410584/1734516580084-01a3b02e-5102-45c0-95db-b78174b59ce7.png)


在alembic/env.py动态获取连接url，并设置到alembic.ini中


alembic/env.py



```
from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config

# Interpret the config file for Python logging.
# This line sets up loggers basically.
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# add your model's MetaData object here
# for 'autogenerate' support
# from myapp import mymodel
# target_metadata = mymodel.Base.metadata
target_metadata = None

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.

import os  # noqa
import sys  # noqa

from server.db.base import Base  # noqa

basedir = os.path.split(os.getcwd())[0]
sys.path.append(basedir)

target_metadata = Base.metadata


def get_url() -> str:
    return os.getenv("DATABASE_URI", "sqlite:///app.db")


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    # url = config.get_main_option("sqlalchemy.url")
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
    connectable = engine_from_config(
        configuration,
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()


```

在 run\_migrations\_offline 中将url获取从配置文件中改成从环境变量中



```
# url = config.get_main_option("sqlalchemy.url")
url = get_url()

```

在 run\_migrations\_online 中修改配置文件的数据库连接信息



```
configuration = config.get_section(config.config_ini_section)
configuration["sqlalchemy.url"] = get_url()
connectable = engine_from_config(
    configuration,
    prefix="sqlalchemy.",
    poolclass=pool.NullPool,
)

```

以上操作之后就能通过服务发现的方式动态使用数据库


# 数据库健康检查


在前面的配置中虽然让服务依赖db,db会先启动然后服务后启动，但是这种情况还会出现数据库连不上的情况，因为db启动不代表服务就绪，未就绪的时候连接会导致connect refuse。解决这个问题的方法是给db增加一个健康检查 healthcheck。



```
services:
  db:
    image: mysql
    container_name: db
    environment:
      - MYSQL_ROOT_PASSWORD=tv_2024 # root用户密码
      - MYSQL_DATABASE=tileView
      - MYSQL_USER=tile_viewer
      - MYSQL_PASSWORD=tv_2024
      - TZ=Asia/Shanghai
    volumes:
      - ./mysql:/var/lib/mysql
      - /etc/localtime:/etc/localtime:ro
    ports:
      - 3306:3306
    restart: always
    healthcheck:
      test: [ "CMD", "mysqladmin", "ping", "-h", "localhost" ]
      interval: 10s
      timeout: 5s
      retries: 3

```

当健康检查完成才代表数据库启动成功，服务才会启动。服务中也需要依赖数据库健康检查完成，写法如下：



```
depends_on:
      db:
        condition: service_healthy

```

 本博客参考[veee加速器](https://blog.liuyunzhuge.com)。转载请注明出处！
