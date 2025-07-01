## Install PostgreSQL 16.5 in Ubuntu 22.04 with Source code 

### Download Source Code 

- https://www.postgresql.org/ftp/source/

### Install Build tools

```bash
sudo apt-get install build-essential libreadline-dev zlib1g-dev flex bison libxml2-dev libxslt-dev libssl-dev
```

### Compile and Install PostgreSQL

```bash
tar -xf postgresql-16.5.tar.gz
cd postgresql-16.5
./configure
make
sudo make install
```

### Initial setting PostgreSQL

- PostgreSQL 전용 사용자 생성

```bash
sudo adduser postgres
```

- DB Data Directory 생성

```bash
export PGDATA=/var/lib/postgresql/16/data
sudo mkdir $PGDATA
sudo chown postgres $PGDATA
```

- DB Cluster Initialize

```bash
sudo -u postgres /usr/local/pgsql/bin/initdb -D $PGDATA
```