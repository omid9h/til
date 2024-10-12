# efficient MySQL import
for large files (more than 2-3 GB) simple importing would be slow. there are some workarounds for that.
- disable some checks temporarily:
```sql
SET autocommit=0;
SET unique_checks=0;
SET foreign_key_checks=0;

-- file content

COMMIT;
SET unique_checks=1;
SET foreign_key_checks=1;
```
- set some MySQL optimization (inside system-databases/mysql):
```sql
SET GLOBAL innodb_buffer_pool_size=1073741824; -- 1GB
SET sql_log_bin=0;
SET GLOBAL max_allowed_packet=1073741824; -- 1GB
```
- if db is inside docker, we can use `pv` (pipe viewer) tool to import the dump file into docker
and see the progress.
```bash
sudo apt-get install pv
```
- the final command would be like this:
```bash
pv ./file-name.sql | docker exec -i container-name mysql -uroot -p123456 --force db-name
```
- use `--force` to ignore errors during import
