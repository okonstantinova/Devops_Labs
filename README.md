## Результат лабораторной работы

Можем подключиться к Grafana, используя данный адрес:
```
http://192.168.0.101:3000
```

Авторизируемся:
![image](https://github.com/user-attachments/assets/9799d27d-5e7d-477b-be53-242ce879c89e)

В разделе Home/Connections/Data sources можем увидеть Prometheus, который собирает метрики:

![image](https://github.com/user-attachments/assets/7a8e7dda-000d-4327-b678-a445cd2d88bd)

Alerting подключён:

![image](https://github.com/user-attachments/assets/bea799e5-53a5-4f7c-abe8-04eb46f9257f)

Присутствует несколько дашбордов, которые были созданы через шаблон:

![image](https://github.com/user-attachments/assets/5240b3ac-cabf-4c74-861e-627b9d7f2f76)

Импортируем их:

![image](https://github.com/user-attachments/assets/219c9f9e-2022-4367-b06b-0b8e4e7a12dd)

Зайдем в один из дашбордов, чтобы посмотреть собранные метрики:

![image](https://github.com/user-attachments/assets/247918c0-d377-4ccf-b22b-30a8296ab833)

Видим, что метрики успешно собираются с сервера
