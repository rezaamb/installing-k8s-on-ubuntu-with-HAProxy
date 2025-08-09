مرور خطاها و راه‌حل‌ها

    ### مشکل HAProxy لاگ

    ارور/هشدار: option httplog not usable ... (needs 'mode http')

    علت: frontend روی TCP بود ولی httplog فقط برای HTTP.

    حل: یا mode http بدیم یا لاگ TCP (option tcplog) بذاریم. ما با TCP ادامه دادیم و هشدار نادیده گرفته شد.

        عدم داشتن DNS برای endpoint

    ارور: host 'K8S-HAproxy' must be a valid IP or RFC-1123 DNS

    علت: نام endpoint در DNS/hosts تعریف نبود.

    حل: اضافه کردن نام‌ها به /etc/hosts روی همه نودها:
```
172.18.42.34 k8s-master
172.18.42.35 k8s-worker01
172.18.42.36 k8s-worker02
172.18.42.37 k8s-lb
```
    
