<div align="center">
  <table>
    <tr>
      <td style="border: none; background: none;">
        <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/0/05/Ansible_Logo.png/480px-Ansible_Logo.png" alt="Ansible Logo" width="100">
      </td>
      <td style="border: none; background: none;">
        <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/e/e3/Openvpn-logo.png/512px-Openvpn-logo.png" alt="OpenVPN Logo" width="150">
      </td>
    </tr>
  </table>
</div>

## ![Lesson](https://img.shields.io/badge/Lesson-otus__vpn-0A84FF?style=for-the-badge&logo=linux&logoColor=white&labelColor=111827)![Author](https://img.shields.io/badge/Author-Kamil%20Ibragimov-10B981?style=for-the-badge&logo=github&logoColor=white&labelColor=111827)![Date](https://img.shields.io/badge/Date-24.12.2025-F59E0B?style=for-the-badge&logo=calendar&logoColor=white&labelColor=111827)

### 📌 Задание
Автоматизация развертывания OpenVPN (TUN/TAP) с помощью **Ansible**:
1. Создание инфраструктуры Server-Client через Vagrant.
2. Настройка OpenVPN сервера и клиента.
3. Проверка пропускной способности канала.

### ✅ Результат
- [x] Автоматический Provisioning всей сети.
- [x] Настроен роутинг и форвардинг трафика.
- [x] Успешные тесты iperf3 (270+ Mbits/sec).

### 🧭 Оглавление
- [🧰 Шаг 1 - Vagrant](#one)
- [🧰 Шаг 2 - Ansible](#two)
- [🧰 Шаг 3 - Проверка](#three)

---

<a id="one"></a>
## 🧰 Шаг 1 - Vagrant
Конфигурация `Vagrantfile` для двух узлов.
* **Server**: 192.168.56.10, проброс порта 1194/udp.
* **Client**: 192.168.56.20.
```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # Настройка сервера
  config.vm.define "server" do |server|
    server.vm.hostname = "server.loc"
    server.vm.network "private_network", ip: "192.168.56.10"
    server.vm.network "forwarded_port", guest: 1194, host: 21194, protocol: "udp"
  end

  # Настройка клиента
  config.vm.define "client" do |client|
    client.vm.hostname = "client.loc"
    client.vm.network "private_network", ip: "192.168.56.20"
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/provision.yml"
    ansible.groups = {
      "vpn_servers" => ["server"],
      "vpn_clients" => ["client"]
    }
    ansible.compatibility_mode = "2.0"
  end
end
```

<a id="two"></a>
## 🧰 Шаг 2 - Ansible
Логика плейбука `provision_ras.yml`.
* **Server**: Настройка `server.conf`, включение IP Forwarding, настройка IPTables.
* **Client**: Сборка `.ovpn` файла из шаблона и скачивание на хост.
```yaml
- name: OpenVPN Server Setup
  hosts: vpn_servers
  become: yes
  tasks:
    - name: Install OpenVPN and Easy-RSA
      apt:
        name: [openvpn, easy-rsa]
        state: present

    - name: Configure IP Forwarding
      sysctl:
        name: net.ipv4.ip_forward
        value: '1'

    - name: NAT for VPN Subnet
      iptables:
        table: nat
        chain: POSTROUTING
        source: 10.10.10.0/24
        out_interface: enp0s3
        jump: MASQUERADE
```

<a id="three"></a>
## 🧰 Шаг 3 - Проверка
Тестирование производительности через `iperf3`.
```bash
# На сервере
iperf3 -s

# На клиенте (через VPN туннель)
iperf3 -c 10.10.10.1 -t 10
```

**Результаты тестов:**
* Пропускная способность (Bitrate): **269 Mbits/sec**.
* Стабильность канала: зафиксировано минимальное количество ретрасмитов (Retr).
