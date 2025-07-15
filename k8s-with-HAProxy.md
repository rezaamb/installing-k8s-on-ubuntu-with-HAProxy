### 🚀🚀 راهنمای گام‌به‌گام برای استقرار یک خوشه Kubernetes روی Ubuntu با HAProxy

به راهنما برای راه‌اندازی یک خوشه Kubernetes با HAProxy خوش‌اومدی! این سند شما رو گام به گام راهنمایی می‌کنه تا HAProxy رو تنظیم کرده و آدرس API server رو درست به‌کار بگیری.

```Tested on Ubuntu 20.04, 22.04 and 24.04 ✅```

✅ این راهنما روی Ubuntu 20.04، 22.04 و 24.04 تست شده و همه مراحل با این نسخه‌ها سازگار هستند.

📝 مقدمه

 کوبرنتیز یک سیستم متن‌باز برای ارکستریشن کانتینر است که کارهای Deployment، مقیاس‌دهی و مدیریت برنامه‌ها رو اتوماتیک می‌کنه. این پروژه توسط گوگل شروع شد و حالا زیر نظر جامعه متن‌باز و Cloud Native Computing Foundation ادامه پیدا می‌کنه.
 
بیاید مرحله به مرحله نصبش کنیم ✔️


## الف. غیرفعال‌سازی swap memory

 کوبرنتیز برای زمان‌بندی کارها، منابع سیستم (RAM واقعی) رو در نظر می‌گیره. اگر Swap فعال باشه، تصمیم‌گیری دقیق زمان‌بندی خراب می‌شه. بنابراین باید قبل از نصب Kubernetes، Swap رو خاموش کنیم. فایل ```/etc/fstab``` رو با ویرایشگر باز کن مثل nano یا vim.

🔹 2 روش برای غیرفعال سازی swap هست:

1.1. روش اول:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

1.2. روش دوم:
```bash
sudo vim /etc/fstab
```

داخل فایل خط مشابه زیر رو پیدا کن:

```
/swapfile          none          swap          sw          0          0
```
و اون خط رو حذف کن، سپس سیستم رو reboot کن.

💡 نکته:

برای proper بودن kubelet، swap باید روی هر دو سرور Master و Worker خاموش بشه.

2️⃣# تنظیم IPv4 Bridge Networking در تمام نودها 🌉

برای مشاهده ترافیک bridged در iptables، یک ماژول و تنظیم sysctl نیاز داریم.

🔹 بارگذاری ماژول‌ها:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

این دو ماژول برای عملکرد صحیح networking کانتینرها الزامن.

🔹 فعال‌سازی bridged traffic:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

🔹 بارگذاری تنظیمات بدون ریبوت:

```bash
sudo sysctl --system
```

3️⃣# نصب Container Runtime (Containerd) 🐳


🔹 نصب containerd:

```bash
sudo apt install containerd -y
```

🔹 تنظیم فایل کانفیگ:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

🔹 ویرایش فایل زیر:


```bash
sudo vim /etc/containerd/config.toml
```

در بخش plugins... گزینه SystemdCgroup = false را به true تغییر بده:

```
SystemdCgroup = true
```

🔹 و برای اعمال تغییرات:

```bash
sudo systemctl restart containerd
```

4️⃣ نصب ابزارهای Kubernetes (kubeadm, kubelet, kubectl) 🛠️

---
بیایید ابزارهای kubelet، kubeadm و kubectl رو روی هر نود (چه master چه worker) نصب کنیم تا بتونیم یک خوشه (cluster) Kubernetes ایجاد کنیم.

این سه مؤلفه برای مدیریت و کار با کلاستر Kubernetes ضروری هستند.
---
و Kubeadm : دستور اصلی برای بوت‌استرپ (راه‌اندازی اولیه) کلاستر.

در واقع kubeadm مسئول ساختن کنترل پلین و تنظیم اولیه‌ی کلاستر هست.
---
و Kubelet : سرویسیه که روی تمام ماشین‌های نود اجرا می‌شه و مسئول اجرا و مدیریت پادها و کانتینرها هست.
---
و Kubectl : ابزار خط فرمان برای تعامل با کلاستر.

با استفاده از kubectl می‌تونی پادها، سرویس‌ها، نودها و بقیه منابع Kubernetes رو بررسی و مدیریت کنی.
---
⚠️ این دستورالعمل‌ها مخصوص نسخه‌ی v1.33 از Kubernetes هستن.
---

4.1 بروزرسانی لیست بسته‌ها:

```bash
sudo apt-get update
```
 apt-transport-https may be a dummy package; if so, you can skip that package

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
4.2 دریافت کلید امضای پکیج Kubernetes:

اگر پوشه‌ای به نام /etc/apt/keyrings روی سیستم وجود نداشته باشه، باید قبل از اجرای دستور curl اون رو بسازی. 
```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
💡 یادداشت:

 در نسخه‌های قدیمی‌تر از Debian 12 و Ubuntu 22.04، این مسیر /etc/apt/keyrings به‌صورت پیش‌فرض وجود نداره.
 
بنابراین باید قبل از اجرای curl، این دایرکتوری رو با دستور mkdir ایجاد کنید.

4.3 افزودن مخزن Kubernetes:
```bash

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4.4 نصب ابزارها:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
4.5 فعال‌سازی kubelet:
```bash
sudo systemctl enable --now kubelet
```

The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.
4.6 فعال‌سازی kubelet:
```bash
sudo systemctl enable kubelet
```
4.7. 4.7 پیش‌کشیدن ایمیج‌ها:
```bash
sudo kubeadm config images pull
```
4.8 راه‌اندازی خوشه (روی Master):

💡 ابتدا HAProxy باید تنظیم باشه! ❗️❗️❗️
```bash
kubeadm init --control-plane-endpoint "apisrv.aranetco.ir:8443" --pod-network-cidr=10.244.0.0/16 --upload-certs
```
⚠️ اگر کلاستر کار نکرد برای ریست :
```bash
sudo kubeadm reset --force
```
برای مدیریت کلاستر، باید kubectl رو روی نود مستر پیکربندی کنی.

اول باید یه دایرکتوری به اسم .kube توی پوشه‌ی Home بسازی.

بعد، فایل پیکربندی ادمین کلاستر (که به صورت پیش‌فرض توی /etc/kubernetes/admin.conf قرار داره) رو کپی کنی داخل این پوشه‌ی شخصی خودت.

و در نهایت، مالکیت اون فایل رو به یوزر فعلی بده تا بتونی بدون sudo از kubectl استفاده کنی:


```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

اگر یوزر روت بودی:
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
5️⃣# نصب Flannel 🌐

  یک روش ساده و راحت برای پیکربندی یک شبکه در لایه ۳ (Layer 3) است که برای کلاستر Kubernetes طراحی شده.

🔸 یعنی چی؟ Flannel کمک می‌کنه نودها و پادها بتونن توی کلاستر با همدیگه ارتباط شبکه‌ای داشته باشن، مخصوصاً وقتی از CNI (Container Network Interface) استفاده می‌کنی.

🔧 دستور نصب Flannel:
```bash
kubectl apply -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

⚠️ هشدار مهم:

اگر از یک podCIDR سفارشی استفاده می‌کنی (غیر از 10.244.0.0/16)، باید فایل بالا رو اول دانلود کنی و بعد داخلش مقدار شبکه رو مطابق با podCIDR خودت تغییر بدی.

یعنی چی؟ مثلاً اگه موقع kubeadm init این CIDR رو دادی:
```bash
--pod-network-cidr=192.168.0.0/16
```
ولی Flannel به صورت پیش‌فرض برای 10.244.0.0/16 تنظیم شده، پس باید فایل kube-flannel.yml رو دستی بگیری و تغییرش بدی

💡 روش درست:

   دانلود فایل:
```bash
curl -O https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml
```

   باز کردن فایل با ویرایشگر:
```bash
vim kube-flannel.yml
```
   پیدا کردن این بخش:
```
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "vxlan"
    }
  }
```
🔁 و تغییرش به چیزی مثل:
```
net-conf.json: |
  {
    "Network": "192.168.0.0/16",
    "Backend": {
      "Type": "vxlan"
    }
  }
```

   حالا نصب:
```bash
kubectl apply -f kube-flannel.yml
```
6️⃣# فعال‌سازی تکمیل خودکار kubectl
BASH:
```bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
ZSH:
```bash
source <(kubectl completion zsh)
echo '[[ $commands[kubectl] ]] && source <(kubectl completion zsh)' >> ~/.zshrc
```
FISH:

For FISH shell, ensure you are using kubectl version 1.23 or above. Then, run the following command:
```bash
echo 'kubectl completion fish | source' > ~/.config/fish/completions/kubectl.fish && source ~/.config/fish/completions/kubectl.fish
```

7️⃣# اضافه کردن Worker به کلاستر 🤝

🔹روی هر نود Worker باید دستور kubeadm join رو اجرا کنی تا این نود به کنترل پلین (همون مستر) متصل بشه.
```bash
sudo kubeadm join [master-node-ip]:8443 --token [token] \
             --discovery-token-ca-cert-hash sha256:[hash]
```

🚀 راهنمای مرحله‌به‌مرحله اضافه کردن نود Worker:

✅ پیش‌نیازها:

🛑 غیرفعال کردن Swap

🌉 فعال‌سازی IPv4 Forwarding و Bridge Networking

🐳 نصب Containerd (به‌عنوان runtime)

🛠️ 🛠️ نصب ابزارهای کوبرنتیز (kubeadm, kubelet, kubectl)

🔗 اجرای دستور kubeadm join برای پیوستن به کلاستر

✅ بررسی اضافه شدن نود Worker

🌐 (اختیاری) ری‌بالانس CoreDNS اگر به‌هم ریخته شده بود

🎉 تبریک! نود Worker شما به کلاستر Kubernetes متصل شده !

---


8️⃣# تنظیم HAProxy 🚀

📝 مقدمه

 که مخفف عبارت High Availability Proxy (پراکسی با در دسترس‌بودن بالا) است،
 
یک نرم‌افزار متن‌باز و بسیار محبوب برای متوازن‌سازی بار (Load Balancing) و پراکسی کردن ترافیک‌های TCP/HTTP می‌باشد.


این برنامه روی سیستم‌عامل‌های لینوکس، مک‌اواس و FreeBSD قابل اجراست.

هدف اصلی HAProxy اینه که کارایی (Performance) و پایداری (Reliability) زیرساخت سرورها رو افزایش بده

با پخش کردن بار کاری بین چندین سرور مثل سرورهای وب، اپلیکیشن یا دیتابیس.


و HAProxy به‌طور گسترده‌ای در محیط‌های مهم و شناخته‌شده استفاده می‌شود

از جمله در شرکت‌ها و سرویس‌هایی مانند GitHub، Imgur، Instagram و Twitter. 🌐



نصب HAProxy:

```bash
apt install haproxy
```
ویرایش ```/etc/haproxy/haproxy.cfg:```
```bash
vim /etc/haproxy/haproxy.cfg
```
Below is the full configuration for HAProxy:

HAProxy Configuration File 🛠️
```
# Frontend for Kubernetes API
frontend k8s-api
  bind *:8443
  mode tcp
  option tcplog
  default_backend k8s-api

# Backend for Kubernetes API
backend k8s-api
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-api-1 192.168.168.51:6443 check

# Monitoring HAProxy
frontend stats
  bind *:8404
  stats enable
  stats uri /stats
  stats refresh 10
```

Explanation of the Configuration File 📝

1. Frontend for Kubernetes API

مورد frontend k8s-api: تعریف یک frontend به‌نام k8s-api برای دریافت ترافیک ورودی.
   
 مورد  bind *:8443:اتصال frontend به پورت 8443 روی همه‌ی اینترفیس‌های شبکه‌ی سرور.(یعنی هر ترافیکی که بیاد روی پورت 8443، وارد این بخش میشه)

 مورد  mode tcp:  این frontend در مد TCP (لایه 4 شبکه) کار می‌کنه. برای ارتباط با Kubernetes API که روی TCP هست، مناسبه

 مورد  option tcplog:  فعال کردن لاگ‌برداری دقیق برای ارتباطات TCP.
   
 مورد default_backend k8s-api:  مشخص می‌کنه که همه‌ی ترافیک‌های دریافتی این frontend باید به backend به‌نام k8s-api ارسال بشن.

3. Backend for Kubernetes API

مورد backend k8s-api: تعریف یک backend به‌نام k8s-api که ترافیک دریافتی از frontend رو هندل می‌کنه.
   
مورد mode tcp: این backend هم در مد TCP کار می‌کنه.
   
مورد option tcplog: فعال‌سازی لاگ‌برداری TCP برای backend.
   
مورد option tcp-check: فعال‌سازی بررسی سلامت سرورها با ارسال TCP Ping. (health check)
   
مورد balance roundrobin: بار ترافیکی رو به‌صورت چرخشی (Round Robin) بین سرورها توزیع می‌کنه
   
مورد default-server: تنظیمات پیش‌فرض برای همه‌ی سرورها:
   
مورد inter 10s: هر ۱۰ ثانیه یک‌بار، health check ارسال می‌شه.
   
مورد downinter 5s:  اگر سرور down بود، health check هر ۵ ثانیه ارسال می‌شه (برای بررسی برگشت به حالت سالم).
   
مورد rise 2: سرور بعد از ۲ بررسی موفق پشت‌سر‌هم به حالت سالم (healthy) میره.
   
مورد fall 2: سرور بعد از ۲ بار شکست پشت‌سر‌هم به حالت ناسالم (unhealthy) میره.
   
مورد slowstart 60s: بعد از سالم شدن سرور، طی ۶۰ ثانیه به‌آرومی وزنش افزایش پیدا می‌کنه (برای جلوگیری از overload ناگهانی).
   
مورد maxconn 250: حداکثر ۲۵۰ ارتباط همزمان به هر سرور اجازه داده میشه.
   
مورد  maxqueue 256:اگر سرور پر بود، حداکثر ۲۵۶ درخواست در صف انتظار می‌مونه.
   
مورد weight 100: وزن سرور ۱۰۰ هست، برای کنترل نسبت بار ترافیکی (موقع load balancing).
  
 معرفی سرور backend به‌نام k8s-api-1 با آی‌پی 192.168.168.51 که روی پورت 6443 گوش میده و health check براش فعاله.

مانیتورینگ HAProxy

مورد frontend stats: تعریف یک frontend جدید به‌نام stats برای دسترسی به پنل مانیتورینگ HAProxy.
    
مورد bind *:8404:  پنل مانیتورینگ روی پورت 8404 در همه‌ی اینترفیس‌های شبکه در دسترسه.
    
مورد stats enable: فعال‌سازی ماژول آماری HAProxy.
    
مورد stats uri /stats:  آدرس دسترسی به پنل آمار رو روی http://<ip>:8404/stats قرار می‌ده.
    
مورد stats refresh 10: هر ۱۰ ثانیه، آمار به‌روز میشه (auto-refresh).

9️⃣ پیکربندی آدرس API Server

When setting up your Kubernetes cluster, it is essential to ensure that the API server address is properly resolved by all machines in the cluster, including the master and worker nodes. This can be achieved in one of two ways:

🌐 1. گزینه اول: DNS سرور (Recommended)

یک رکورد A‌ بساز که آدرس دامنه‌ی API مثل api-server.example.com رو به IP سرور API نگاشت کنه.

If you have a DNS server configured, you should add the API server's address to the DNS records. This allows all nodes and clients in the cluster to resolve the API server's domain name to its IP address automatically.

For example, you would create a DNS record that maps the API server's domain name (e.g., api-server.example.com) to its IP address (e.g., 192.168.1.50).

✅ Advantages of Using DNS:

    Centralized domain name resolution.
    Simplifies management, especially in larger or dynamic environments.

🖥️ 2.گزینه دوم: فایل /etc/hosts

If you do not have a DNS server, you must manually configure the API server's domain name resolution by adding an entry to the /etc/hosts file on all machines associated with the Kubernetes cluster, including the master and worker nodes.
Steps to Configure /etc/hosts:

روی هر نود خط زیر رو اضافه کن:
```bash
sudo vim /etc/hosts
```
Add the following line to map the API server's IP address to its domain name:
```
192.168.1.50 api-server.example.com
```
        Replace 192.168.1.50 with the IP address of your API server.
        Replace api-server.example.com with the desired domain name.

    Save and close the file.

❗ Important Note

When you are not using a DNS server, you must include the domain name and API address of the API server in the /etc/hosts file on all machines in the Kubernetes cluster, including the master and worker nodes. This ensures that all nodes can resolve the API server's domain name to its IP address.
📋 Why This Step is Important

    DNS Server: Using a DNS server is the preferred method because it centralizes domain name resolution and simplifies management, especially in larger or dynamic environments.
    /etc/hosts File: This method is suitable for smaller setups or testing environments but requires manual updates on each machine if the API server's IP address changes.

🎉 نتیجه‌گیری
حالا HAProxy و API سرور با پورت 8443 تنظیم شدن، Kubernetes در مسیر راه‌اندازی هست، و آماده راه‌اندازی اولیه هستی 🚀

🌟 Why Use Port 8443 for HAProxy Instead of 6443 for the Kubernetes API Server?

When setting up a Kubernetes cluster with HAProxy as a load balancer, you may notice that the Kubernetes API server listens on port 6443 by default, but HAProxy is configured to listen on port 8443. This is a deliberate and important design choice. Here's why:
🔧 1. Separation of Responsibilities

    Port 6443 is the default port used by the Kubernetes API server for secure communication.
    HAProxy acts as an intermediary (load balancer) between clients (e.g., kubectl, Kubernetes components, or external users) and the API server.
    To avoid confusion and clearly distinguish between direct API server access and load-balanced access, HAProxy is configured to listen on port 8443.

🚫 2. Avoiding Port Conflicts

    If HAProxy were configured to listen on port 6443, it would conflict with the Kubernetes API server, which is already bound to that port on the master node(s).
    By using port 8443, we ensure that both services (HAProxy and the Kubernetes API server) can coexist without any port binding issues.

🌐 3. HAProxy as a Gateway

    HAProxy serves as a single entry point for all incoming traffic to the Kubernetes API server.
    By assigning it a different port (8443), it becomes clear that traffic on this port is being routed through the load balancer.
    This abstraction simplifies client configuration, as clients only need to know the HAProxy endpoint (e.g., https://haproxy.example.com:8443) rather than managing multiple API server endpoints.

🔒 4. Security and Access Control

    Using a different port for HAProxy allows for better access control and firewall rules:
        Port 6443 can be restricted to internal components or administrators who need direct access to the API server.
        External users or clients can be directed to port 8443, where HAProxy handles load balancing and potentially additional security measures (e.g., rate limiting, logging).

⚖️ 5. Flexibility in Multi-Master Setups

    In a multi-master Kubernetes cluster, HAProxy distributes traffic across multiple API servers running on different master nodes.
    By using port 8443, HAProxy provides a single, unified entry point for clients, while internally forwarding requests to the appropriate master node on port 6443.

🛠️ How It Works in Your Configuration

Here’s how this setup is reflected in your HAProxy configuration:

# Frontend for Kubernetes API
frontend k8s-api
  bind *:8443  # HAProxy listens on port 8443 for incoming traffic
  mode tcp
  option tcplog
  default_backend k8s-api

# Backend for Kubernetes API
backend k8s-api
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  server k8s-api-1 192.168.1.50:6443 check  # Forwards traffic to the API server on port 6443

    Frontend (bind *:8443): HAProxy listens for incoming traffic on port 8443.
    Backend (192.168.1.50:6443): HAProxy forwards the traffic to the Kubernetes API server running on port 6443.

This setup ensures that:

    Clients connect to HAProxy on port 8443.
    HAProxy distributes the traffic to the API server(s) on port 6443.

🎯 Key Benefits of Using Port 8443 for HAProxy

    No Port Conflicts: Avoids conflicts with the Kubernetes API server, which uses port 6443.
    Clear Separation: Distinguishes between direct API server access and load-balanced access.
    Simplified Access: Provides a single, unified entry point for clients.
    Enhanced Security: Allows for better access control and firewall rules.
    Scalability: Supports multi-master setups by routing traffic to multiple API servers.

🎉 Conclusion

Using port 8443 for HAProxy instead of port 6443 is a best practice that ensures a clean, scalable, and secure architecture for your Kubernetes cluster. It avoids conflicts, simplifies management, and provides a clear separation of responsibilities between HAProxy and the Kubernetes API server. 🚀
Author ✍️
Created by Reza BND. If you find this repository helpful, feel free to fork it or contribute!
