``` مشکل HAProxy لاگ```

 ارور/هشدار: option httplog not usable ... (needs 'mode http')

علت: frontend روی TCP بود ولی httplog فقط برای HTTP.

حل: یا mode http بدیم یا لاگ TCP (option tcplog) بذاریم. ما با TCP ادامه دادیم و هشدار نادیده گرفته شد.

``` عدم داشتن DNS برای endpoint```

ارور: host 'K8S-HAproxy' must be a valid IP or RFC-1123 DNS

علت: نام endpoint در DNS/hosts تعریف نبود.

حل: اضافه کردن نام‌ها به /etc/hosts روی همه نودها:
```
172.18.42.34 k8s-master
172.18.42.35 k8s-worker01
172.18.42.36 k8s-worker02
172.18.42.37 k8s-lb
```

``` در kubeadm init گیر روی healthz و deadline```

ارور: context deadline exceeded / unexpected EOF
علت اصلی: درخواست‌های لوکال به LB از طریق HTTP(S)_PROXY می‌رفت.
تشخیص: curl -k https://k8s-lb:8443/healthz → فقط با --noproxy یا بعد از تنظیم no_proxy جواب می‌داد.
حل: تنظیم/پاک‌کردن پراکسی محیطی:
```
export no_proxy=127.0.0.1,localhost,10.96.0.0/12,10.244.0.0/16,172.18.42.0/28,172.18.42.34,172.18.42.37,k8s-lb
export NO_PROXY=$no_proxy
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
```

``` در kubectl با admin.conf دسترسی نداشت```
ارور: User "kubernetes-admin" cannot list nodes ...
علت: نبودن ClusterRoleBinding برای گروه kubeadm:cluster-admins (روی این چرخه init).
حل: ساخت CRB:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: kubeadm:cluster-admins }
roleRef: { apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: cluster-admin }
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: kubeadm:cluster-admins

```

``` در CoreDNS Pending و نود Master NotReady ```

ارورهای نود: Network plugin returns error: cni plugin not initialized

علت: CNI روی مستر کامل init نشده/پراکسی مزاحم. Flannel فقط روی مستر بود؛ PodCIDR ست شد ولی kubelet نتورک رو بالا نمی‌آورد.

حل‌ها:

 نصب و اعمال Flannel:
 ```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
اطمینان از وجود باینری‌های CNI:
```
apt-get install -y containernetworking-plugins
ls /opt/cni/bin
```
پاک‌کردن کش CNI و ری‌استارت:
```
rm -rf /var/lib/cni/* /var/lib/kubelet/network/*
systemctl restart kubelet
```

``` در kubeadm reset و بازسازی ```

ارور: بعد از reset، the CA files do not exist هنگام kubeadm init phase kubeconfig admin

علت: reset فایل‌های PKI را پاک کرده بود.

حل: اجرای فازهای certs و control-plane و kubeconfig طبق ترتیب:
```
kubeadm init \
  --control-plane-endpoint k8s-lb:8443 \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs
```

``` مشکل دائمی پراکسی روی سرویس‌ها (گلوگاه اصلی!) ```

علامت: systemctl show kubelet -p Environment → HTTP_PROXY/HTTPS_PROXY ست بود.

علت: Drop-in های systemd پراکسی را روی kubelet/containerd فعال کرده بودند.

حل نهایی (روی مستر و در صورت نیاز Workerها):
```
vim /etc/systemd/system/kubelet.service.d/proxy.conf
```
محتوا:
```
[Service]
Environment="HTTP_PROXY="
Environment="HTTPS_PROXY="
Environment="NO_PROXY=127.0.0.1,localhost,.svc,.cluster.local,10.96.0.0/12,10.244.0.0/16,172.18.42.0/28,172.18.42.34,172.18.42.35,172.18.42.36,172.18.42.37,k8s-lb"
Environment="no_proxy=127.0.0.1,localhost,.svc,.cluster.local,10.96.0.0/12,10.244.0.0/16,172.18.42.0/28,172.18.42.34,172.18.42.35,172.18.42.36,172.18.42.37,k8s-lb"
```
(برای containerd هم اگر داشتی، مشابهش را خالی کن)

سپس:
```
systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
```
و راستی‌آزمایی:
```
systemctl show kubelet -p Environment
```

``` خطای Join روی Worker ```

ارور:
```
[ERROR FileAvailable--etc-kubernetes-kubelet.conf]: ... exists
[ERROR Port-10250]: Port 10250 is in use
[ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: ... exists
```
علت: تلاش Join روی نودی که قبلا نیمه‌کاره کانفیگ شده بود.

حل: ریست تمیز نود و Join دوباره:
```
kubeadm reset -f
rm -rf /etc/cni/net.d /var/lib/cni/* /var/lib/kubelet/*
systemctl restart kubelet
kubeadm join <k8s-lb:8443> --token <...> --discovery-token-ca-cert-hash <...>
```
``` دانلود kubectl دستی و checksum mismatch ```

ارور: sha256sum ... did NOT match

علت: فایل باینری نبود (فقط sha256 دانلود شده بود) یا mismatch مسیر/پراکسی.

حل: نصب رسمی از ریپو pkgs.k8s.io (apt)، که انجام شد و نسخه 1.33.3 نصب شد.
