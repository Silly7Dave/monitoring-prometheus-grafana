# Monitoring klastra Kubernetes i routera z wykorzystaniem Prometheus i Grafana
Projekt przedstawia wykorzystanie narzędzi Prometheus, Grafana, kube-state-metrics i SNMP Exporter do kompleksowego monitoringu infrastruktury, w skład której wchodzą:
- Klastry Kubernetes na serwerach Kratos i Fujitsu
- Router MikroTik
- Centralny serwer monitorujący Atlas
- Terminal wizualizacji danych – Raspberry Pi Zero 2W
## Cele projektu
- **Monitoring klastra Kubernetes**

  Monitorowanie klastra złożonego z 12 węzłów (Kratos) i 1 mastera (Fujitsu) za pomocą kube-state-metrics.

- **Zbieranie danych z routera (Mikrotik) przez SNMP**

  Konfiguracja SNMP Exportera, który udostępnia dane dla Prometheusa.

- **Centralizacja monitoringu na serwerze Atlas**

  Atlas pełni funkcję agregatora danych z klastra i routera, z zainstalowanymi Prometheus + Grafana.

- **Prezentacja dashboardu na Raspberry Pi**

  Raspberry Pi Zero 2W automatycznie uruchamia Grafana Kiosk i prezentuje wybrany dashboard na ekranie.
## Topologia
<img width="3942" height="3714" alt="image" src="https://github.com/user-attachments/assets/4687f0c8-1e72-4172-867e-aa8409e61837" />
Grafika ilustruje architekturę projektu, w tym rozmieszczenie serwerów, routera i urządzenia wyświetlającego dashboard. Pokazano także przepływ danych między komponentami.

## Wykorzystane urządzenia i technologie

| Urządzenie               | Opis                                                             |
| ------------------------ | ---------------------------------------------------------------- |
| **Atlas**                | Linux Debian, TrueNAS, Prometheus, Grafana – agregacja danych    |
| **Kratos**               | Debian + Proxmox VE, worker nodes Kubernetes, kube-state-metrics |
| **Fujitsu**              | Debian + Proxmox VE, master node Kubernetes                      |
| **Router**               | MikroTik RB5009UG+S+, monitoring przez SNMP                      |
| **Raspberry Pi Zero 2W** | Raspbian + Grafana Kiosk – wyświetlanie dashboardu               |

# Instalacja Kubernetes (Debian)
## Przygotowanie systemu
```
sudo apt update && sudo apt upgrade
sudo apt install -y curl apt-transport-https
```
## Instalacja Dockera
- Dodanie Klucza GPG Dockera:
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
- Dodanie Repozytorium Dockera:
```
echo "deb [signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] 	https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo 	tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- Instalacja Dockera:
```
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
```
## Instalacja kubectl, kubelet, kubeadm
```
sudo apt install -y kubectl kubelet kubeadm
```
## Inicjalizacja klastra
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## Dołączanie węzłów roboczych
Na każdym workerze:
```
kubeadm join ...
```
- Weryfikacja Dołączenia Węzłów Roboczych:
```
kubectl get nodes
```
# Konfiguracja Prometheus
>⚠️ Instalacja Prometheusa została pominięta. Szczegóły znajdziesz tutaj:  
>🔗 https://prometheus.io/docs/prometheus/latest/installation/

Zakładamy działający Prometheus pod http://192.168.0.111:30002.

prometheus.yml
```
scrape_configs:
    - job_name: 'Mikrotik'
    static_configs:
        - targets:
            - 192.168.0.1 # Mikrotik device
    metrics_path: /snmp
    params:
        module: [mikrotik]
    relabel_configs:
        - source_labels: [__address__]
            target_label: __param_target
        - source_labels: [__param_target]
            target_label: instance
        - target_label: __address__
            replacement: 192.168.0.111:9116 # SNMP Exporter
    - job_name: 'kubernetes'
        static_configs:
            - targets: ['192.168.0.200:30000'] # kube-state-metrics
```
# Konfiguracja Grafana
>⚠️ Instalacja Grafany została pominięta. Szczegóły znajdziesz tutaj:  
>🔗 https://grafana.com/docs/grafana/latest/setup/installation/

Zakładamy działającą instancję Grafany pod http://192.168.0.111:30001.

## Dodanie źródła danych (Prometheus)
1. Logowanie do Grafany

2. Configuration → Data Sources → Add data source

3. Wybierz Prometheus

4. Wpisz URL: http://192.168.0.111:30002

5. Kliknij Save & Test

## Import dashboardu (ID: 13332 – Node Exporter Full)
1. Dashboards → Import

2. Wprowadź ID: 13332

3. Wybierz źródło danych: Prometheus

4. Kliknij Import

# Instalacja kube-state-metrics z NodePort
- Pobranie kube-state-metrics:
```
git clone https://github.com/kubernetes/kube-state-metrics.git
cd kube-state-metrics
```

prometheus.yml
```
      kind: Service
	apiVersion: v1
	metadata:
	  namespace: kube-system
	  name: kube-state-nodeport
	spec:
	  selector:
	    app.kubernetes.io/name: kube-state-metrics
	  ports:
	  - protocol: TCP
	    port: 8080
	    nodePort: 30000
	  type: NodePort
```

```
kubectl apply -f kube-state-metrics-nodeport.yaml
```

# Raspberry Pi Zero 2W – Grafana Kiosk
## Instalacja systemu
- Zainstaluj Raspbian (np. za pomocą Raspberry Pi Imager)

## Instalacja Grafana Kiosk
```
cd ~
git clone https://github.com/grafana/grafana-kiosk.git
sudo cp bin/grafana-kiosk.linux.armv7 /usr/bin/grafana-kiosk
```
## Autostart (/etc/xdg/lxsession/LXDE-pi/autostart)
```
@lxpanel --profile LXDE-pi
@pcmanfm --desktop --profile LXDE-pi
@xscreensaver -no-splash
@/usr/bin/grafana-kiosk \
  -URL=http://192.168.0.111:30037/d/af06c875-0544-4d5b-9f30-2a9b5cee5ac7/raspberry-pi-zero \
  -login-method=local \
  -username=YOUR_USERNAME \
  -password=YOUR_PASSWORD \
  -kiosk-mode=full \
  -lxde
```

# Przydatne linki i źródła
- [Setting up Prometheus and Grafana Outside Kubernetes cluster - continuous learner](https://directdevops.blog/2021/09/09/setting-up-prometheus-and-grafana-outside-kubernetes-cluster/)
- [Monitor Mikrotik Router with Prometheus and Grafana on Ubuntu Server - Ripon4You](https://www.youtube.com/watch?v=9V4S1k3nZGg)
