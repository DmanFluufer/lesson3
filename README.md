## Описание стенда

В качестве отслеживаемой системы я выбрал ВМ с тестового стенда компании, на котором стоит один из продуктов - PCRF, который является узлом управления политиками и тарификации в сетях связи. Сам Prometheus также стоит на отдельном тестовом сервере. На ВМ с отслеживаемой системой я поставил:

- node_exporter
- postgres_exporter
- jmx_exporter
- blackbox_exporter

## Установка и настройка Prometheus

1. Скачиваем архив

   ```shell
   wget https://github.com/prometheus/prometheus/releases/download/v2.54.0/prometheus-2.54.0.linux-amd64.tar.gz
   ```

2. Создаем пользователя и нужные каталоги, настраиваем для них владельцев 

   ```shell
   useradd --no-create-home --shell /bin/false prometheus
   mkdir /etc/prometheus
   mkdir /var/lib/prometheus
   chown prometheus:prometheus /etc/prometheus
   chown prometheus:prometheus /var/lib/prometheus
   ```

3. Распаковываем архив, для удобства переименовываем директорию и копируем бинарники в /usr/local/bin 

   ```shell
   tar -xvzf prometheus-2.54.0.linux-amd64.tar.gz
   mv prometheus-2.54.0.linux-amd64 prometheuspackage
   cp prometheuspackage/prometheus /usr/local/bin/
   cp prometheuspackage/promtool /usr/local/bin/
   ```

4. Меняем владельцев у бинарников

   ```shell
   chown prometheus:prometheus /usr/local/bin/prometheus
   chown prometheus:prometheus /usr/local/bin/promtool
   ```

5. По аналогии копируем библиотеки

   ```shell
   cp -r prometheuspackage/consoles /etc/prometheus
   cp -r prometheuspackage/console_libraries /etc/prometheus
   chown -R prometheus:prometheus /etc/prometheus/consoles
   chown -R prometheus:prometheus /etc/prometheus/console_libraries
   ```

6. Создаем файл конфигурации и добавляем туда джобы для будущих экспортеров

   ```yaml
   global:
    scrape_interval: 10s
   
   scrape_configs:
    - job_name: 'prometheus_master'
      scrape_interval: 5s
      static_configs:
        - targets: ['localhost:9090']
    - job_name: 'pcrf_node_exporter_centos'
      scrape_interval: 5s
      static_configs:
        - targets: ['192.168.110.120:9100']
    - job_name: 'pcrf_postgres_exporter'
      static_configs:
        - targets: ['192.168.110.120:9187']
    - job_name: 'pcrf_jmx_eporter'
      static_configs:
        - targets: ['192.168.110.120:9011']
    - job_name: 'blackbox_http'
      metrics_path: /probe
      params:
        module: [http_2xx]
      static_configs:
        - targets:
          - http://192.168.110.120:8080/pcrf.admin
      relabel_configs:
        - source_labels: [__address__]
          target_label: __param_target
        - source_labels: [__param_target]
          target_label: instance
        - target_label: __address__
          replacement: 192.168.110.120:9115
   ```

7. Создаём systemd сервис и запускаем Prometheus

   ```shell
   $ vim /etc/systemd/system/prometheus.service
   [Unit]
   Description=Prometheus
   Wants=network-online.target
   After=network-online.target
   
   [Service]
   User=prometheus
   Group=prometheus
   Type=simple
   ExecStart=/usr/local/bin/prometheus \
   	--config.file /etc/prometheus/prometheus.yml \
   	--storage.tsdb.path /var/lib/prometheus/ \
   	--web.console.templates=/etc/prometheus/consoles \
   	--web.console.libraries=/etc/prometheus/console_libraries
   
   [Install]
   WantedBy=multi-user.target
   
   $ systemctl daemon-reload
   $ systemctl start prometheus
   $ systemctl status prometheus
   ```

   

## Установка и настройка экспортеров на удалённый сервер

1. Опустим момент загрузки node_exporter и подготовки окружения и сразу перейдём к созданию сервиса

   ```
   [Unit]
   Description=Node Exporter
   After=network.target
   
   [Service]
   User=nodeusr
   Group=nodeusr
   Type=simple
   ExecStart=/usr/local/bin/node_exporter
   
   [Install]
   WantedBy=multi-user.target
   ```

2. Делаем релоад и запускаем node_exporter

   ```shell
   systemctl daemon-reload
   systemctl start node_exporter
   systemctl enable node_exporter
   ```

3. Проверяем работу экспортера

   ```shell
   curl "192.168.110.120:9100/metrics"
   
   # HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
   # TYPE go_gc_duration_seconds summary
   go_gc_duration_seconds{quantile="0"} 5.2558e-05
   go_gc_duration_seconds{quantile="0.25"} 7.0356e-05
   go_gc_duration_seconds{quantile="0.5"} 9.0455e-05
   go_gc_duration_seconds{quantile="0.75"} 0.000175149
   go_gc_duration_seconds{quantile="1"} 0.000408153
   go_gc_duration_seconds_sum 3.196127002
   go_gc_duration_seconds_count 35971
   # HELP go_goroutines Number of goroutines that currently exist.
   # TYPE go_goroutines gauge
   go_goroutines 8
   # HELP go_info Information about the Go environment.
   # TYPE go_info gauge
   go_info{version="go1.19.3"} 1
   ...
   ```

4. Создаём сервис для postgres_exporter

   ```
   [Unit]
   Description=PostgreSQL Exporter
   Wants=network-online.target
   After=network-online.target
   
   [Service]
   User=postgres
   Group=postgres
   EnvironmentFile=/path/to/exporter/postgres_exporter/postgres_exporter.env
   ExecStart=/usr/local/bin/postgres_exporter \
       --web.listen-address=:9187 \
       --web.telemetry-path=/metrics
   Restart=always
   
   [Install]
   WantedBy=multi-user.target
   ```

5. Создаём файл окружения для postgres_exporter

   ```
   DATA_SOURCE_NAME="postgresql://user:password@localhost:5432/postgres?sslmode=disable"
   ```

6. Делаем релоад и запускаем

   ```shell
   systemctl daemon-reload
   systemctl start postgres_exporter
   systemctl enable postgres_exporter
   ```

7. Проверяем работу жкспортера

   ```shell
   curl "192.168.110.120:9187/metrics" | head -n 20
   
   go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
   # TYPE go_gc_duration_seconds summary
   go_gc_duration_seconds{quantile="0"} 6.4754e-05
   go_gc_duration_seconds{quantile="0.25"} 0.000134986
   go_gc_duration_seconds{quantile="0.5"} 0.000202948
   go_gc_duration_seconds{quantile="0.75"} 0.00033427
   go_gc_duration_seconds{quantile="1"} 0.074042552
   go_gc_duration_seconds_sum 27.034912498
   go_gc_duration_seconds_count 31814
   # HELP go_goroutines Number of goroutines that currently exist.
   # TYPE go_goroutines gauge
   go_goroutines 11
   # HELP go_info Information about the Go environment.
   # TYPE go_info gauge
   go_info{version="go1.21.3"} 1
   # HELP go_memstats_alloc_bytes Number of bytes allocated and still in use.
   # TYPE go_memstats_alloc_bytes gauge
   go_memstats_alloc_bytes 2.225824e+06
   ...
   ```

8. Поскольку отслеживаемое приложение работает на Java, то я решил добавить jmx_exporter. Создаём файл конфигурации для jmx_exporter. Нужные атрибуты я подтянул из уже готовой конфигурации для Zabbix

   ```yaml
   startDelaySeconds: 0
   ssl: false
   lowercaseOutputName: false
   lowercaseOutputLabelNames: false
   
   rules:
     - pattern: 'java.lang<type=ClassLoading><attribute=LoadedClassCount>'
       name: jvm_classloading_loaded_classes
       type: GAUGE
       help: Number of loaded classes in the JVM
   
     - pattern: 'java.lang<type=ClassLoading><attribute=TotalLoadedClassCount>'
       name: jvm_classloading_total_loaded_classes
       type: COUNTER
       help: Total number of classes loaded since the JVM started
   
     - pattern: 'java.lang<type=ClassLoading><attribute=UnloadedClassCount>'
       name: jvm_classloading_unloaded_classes
       type: COUNTER
       help: Number of unloaded classes in the JVM
   ...
   ```

9. Добавляем аргументы для запуска приложения

   ```bash
   -Dcom.sun.management.jmxremote \
   -Dcom.sun.management.jmxremote.port=9010 \
   -Dcom.sun.management.jmxremote.rmi.port=9010 \
   -Dcom.sun.management.jmxremote.local.only=false \
   -Dcom.sun.management.jmxremote.authenticate=false \
   -Dcom.sun.management.jmxremote.ssl=false \
   -javaagent:/var/protei/ddd/jmx_exporter/jmx_prometheus_javaagent-1.0.1.jar=9011:/path/to/jmx_exporter/config.yaml \
   ```

10. Проверяем работу экспортера

    ```shell
    curl "192.168.110.120:9011/metrics" | head -n 20
    
    100 11120  100 11120    0     # HELP jmx_config_reload_failure_total Number of times configuration have failed to be reloaded.
    # TYPE jmx_config_reload_failure_total counter
    jmx_config_reload_failure_total 0.0
    # HELP jmx_config_reload_success_total Number of times configuration have successfully been reloaded.
    # TYPE jmx_config_reload_success_total counter
    jmx_config_reload_success_total 0.0
    # HELP jmx_exporter_build_info JMX Exporter build information
    # TYPE jmx_exporter_build_info gauge
    jmx_exporter_build_info{name="jmx_prometheus_javaagent",version="1.0.1"} 1
    # HELP jmx_scrape_cached_beans Number of beans with their matching rule cached
    # TYPE jmx_scrape_cached_beans gauge
    jmx_scrape_cached_beans 0.0
    # HELP jmx_scrape_duration_seconds Time this JMX scrape took, in seconds.
    # TYPE jmx_scrape_duration_seconds gauge
    jmx_scrape_duration_seconds 0.031447689
    # HELP jmx_scrape_error Non-zero if this scrape failed.
    # TYPE jmx_scrape_error gauge
    jmx_scrape_error 0.0
    ...
    ```

11. Перейдём к созданию сервиса для blackbox_exporter

    ```
    [Unit]
    Description=Blackbox Exporter
    Wants=network-online.target
    After=network-online.target
    
    [Service]
    User=blackbox_exporter
    Group=blackbox_exporter
    Type=simple
    ExecStart=/usr/local/bin/blackbox_exporter \
      --config.file=/path/to/blackbox_exporter/comfig.yml
    
    [Install]
    WantedBy=multi-user.target
    ```

12. Создаём конфигурацию, которая будет следить за HTTP кодами

    ```yaml
    modules:
      http_2xx:
        prober: http
        timeout: 5s
        http:
          valid_http_versions: ["HTTP/1.1", "HTTP/2"]
          valid_status_codes: []
          follow_redirects: true
          fail_if_ssl: false
          fail_if_not_ssl: false
    ```

13. Делаем релоад и запускаем

    ```shell
    systemctl daemon-reload
    systemctl start node_exporter
    systemctl enable node_exporter
    ```

14. Проверяем работу экспортера

    ```shell
    curl "192.168.110.120:9115/metrics" | head -n 20
    
    blackbox_exporter_build_info A metric with a constant '1' value labeled by version, revision, branch, goversion from which blackbox_exporter was built, and the goos and goarch for the build.
    # TYPE blackbox_exporter_build_info gauge
    blackbox_exporter_build_info{branch="HEAD",goarch="amd64",goos="linux",goversion="go1.22.2",revision="ef3ff4fef195333fb8ee0039fb487b2f5007908f",tags="unknown",version="0.25.0"} 1
    # HELP blackbox_exporter_config_last_reload_success_timestamp_seconds Timestamp of the last successful configuration reload.
    # TYPE blackbox_exporter_config_last_reload_success_timestamp_seconds gauge
    blackbox_exporter_config_last_reload_success_timestamp_seconds 1.7236358207132034e+09
    # HELP blackbox_exporter_config_last_reload_successful Blackbox exporter config loaded successfully.
    # TYPE blackbox_exporter_config_last_reload_successful gauge
    blackbox_exporter_config_last_reload_successful 1
    # HELP blackbox_module_unknown_total Count of unknown modules requested by probes
    # TYPE blackbox_module_u--:-- nknown_total counter
    -blackbox_module_unknown_total 0
    -# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
    :# TYPE go_gc_duration_seconds summary
    -go_gc_duration_seconds{quantile="0"} 5.4427e-05
    -go_gc_duration_seconds{quantile="0.25"} 0.000515466
    :go_gc_duration_seconds{quantile="0.5"} 0.000731424
    -go_gc_duration_seconds{quantile="0.75"} 0.000887395
    -go_gc_duration_seconds{quantile="1"} 0.008883122
     go_gc_duration_seconds_sum 0.056040402
     ...
    ```



В целом я выбрал отслеживание именно этих параметров, поскольку веб у отслеживаемого продукта является лишь удобным инструментом управления, но главная часть - это бэкенд, поэтому ему уделено больше внимания
