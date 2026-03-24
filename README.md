# Raspberry Pi5-Edge-M.2-Coral_TPU
## Полное руководство по установке драйверов gasket и apex для Google Coral M.2 TPU на Raspberry Pi 5
### Оглавление
1. Подготовка системы
2. Проверка подключения PCIe
3. Настройка параметров загрузки
4. Сборка драйвера через gasket-builder
5. Установка собранного пакета
6. Загрузка модулей и настройка автозагрузки
7. Настройка прав доступа
8. Перезагрузка и финальная проверка
9. Установка pyenv и виртуального окружения 
10. Установка Python 3.9.16
11. Установка CoralTPU и финальное тестирование

### 1.Подготовка системы
Перед началом убедитесь, что система и установлены необходимые пакеты:
```bash
sudo apt update #&& sudo apt full-upgrade -y
sudo apt install -y git curl wget build-essential dkms linux-headers-$(uname -r) docker.io
```
Добавьте пользователя в группу docker, чтобы использовать Docker без sudo:
```bash
sudo usermod -aG docker $USER
newgrp docker
```
### 2.Проверка подключения PCIe  
Убедитесь, что модуль Coral определяется системой:
```bash
lspci -nn | grep 1ac1
```
Ожидаемый вывод:
```bash
0000:01:00.0 System peripheral [0880]: Global Unichip Corp. Coral Edge TPU [1ac1:089a]
```
### 3.Настройка параметров загрузки  
Добавьте необходимые параметры в config.txt:
```bash
sudo nano /boot/firmware/config.txt
```
Добавьте в конец файла:
```
[all]
dtparam=pciex1
dtparam=pciex1_gen=3
dtparam=pciex1=on
dtoverlay=pineboards-hat-ai
dtoverlay=pcie-32bit-dma
kernel=kernel8.img
dtoverlay=coral-msi-fix
```
И в cmdline.txt:
```
sudo nano /boot/firmware/cmdline.txt
```
В конец существующей строки добавьте параметр:
```
pcie_aspm=off
```
После внесения изменений перезагрузите Raspberry Pi:
```
sudo reboot
```
Убедитесь, что Docker установлен и работает:
```
sudo systemctl enable --now docker
docker –version
```
### 4.Сборка драйвера Gasket
Установка инструментов для сборки 
```
sudo apt-get update
sudo apt-get install -y git build-essential devscripts debhelper dh-dkms
```
Скачивание и ПАТЧИНГ драйвера (Самое важное! Выполняйте команды по одной):
```
# 1. Скачиваем исходный код драйвера
cd ~
rm -rf gasket-driver  # удаляем если был старый
git clone https://github.com/google/gasket-driver.git
cd gasket-driver
# 2. ПРИМЕНЯЕМ ПАТЧ (Исправляем ошибку class_create)
# Эта команда меняет устаревший вызов функции на новый, совместимый с Kernel 6.6
sed -i 's/class_create(THIS_MODULE, "gasket")/class_create("gasket")/' src/gasket_core.c
# 3. Собираем установочный пакет (.deb)
# Это займет 1-2 минуты
debuild -us -uc -tc –b
```
6.Установка собранного драйвера:
```
cd ..
# Устанавливаем наш свежий драйвер
sudo dpkg -i gasket-dkms_*.deb
# Устанавливаем библиотеку (она теперь встанет нормально)
sudo apt-get install -y libedgetpu1-std
```
Создание файла coral-msi-fix.dts
```
cat <<EOF > coral-msi-fix.dts
/dts-v1/;
/plugin/;
/ {
    compatible = "brcm,bcm2712";
    fragment@0 {
        target = <&pcie1>;
        __overlay__ {
            msi-parent = <&pcie1>;
        };
    };
};
EOF
```
Компиляция в бинарный файл (Теперь превратим этот текст в понятный ядру формат .dtbo):
```
# Убедимся, что компилятор установлен
sudo apt-get install -y device-tree-compiler
# Компилируем
dtc -I dts -O dtb -o coral-msi-fix.dtbo coral-msi-fix.dts
```
Установка файла в систему (копируем полученный файл в папку с оверлеями ядра):
```
sudo cp coral-msi-fix.dtbo /boot/firmware/overlays/
```
Перезагрузите:
```
sudo reboot
```
После перезагрузки проверьте:
```
dmesg | grep apex
```
Драйвер должен заработать без ошибки «-28».  
Загрузка модулей и настройка автозагрузки:
```
sudo modprobe gasket
sudo modprobe apex
```
Проверьте наличие устройства:
```
ls /dev/apex*
```
Должен появиться /dev/apex_0  
Настройте автоматическую загрузку модулей при старте:
```
echo "gasket" | sudo tee /etc/modules-load.d/gasket.conf
echo "apex" | sudo tee -a /etc/modules-load.d/gasket.conf
```
Настройка прав доступа  
Создайте udev-правило для корректных прав на устройство:
```
sudo sh -c 'echo "SUBSYSTEM==\"apex\", MODE=\"0660\", GROUP=\"apex\"" > /etc/udev/rules.d/65-apex.rules'
sudo groupadd -r apex 2>/dev/null || true
sudo usermod -a -G apex $USER
sudo udevadm control --reload-rules
sudo udevadm trigger
```
Перезагрузка и финальная проверка:
```
sudo reboot
```
После перезагрузки проверьте:
```
ls -l /dev/apex*               # должно быть /dev/apex_0
lsmod | grep -E "gasket|apex"  # модули должны быть загружены
dmesg | grep -i apex | tail -10 # сообщения об инициализации
```
Установка Pyenv и python 3.9.16  
Установка pyenv и зависимостей:
```
sudo apt update
sudo apt install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev libffi-dev liblzma-dev python3-openssl git
curl https://pyenv.run | bash
```
Проверьте, что папка ~/.pyenv существует:
```
ls -la ~/.pyenv/
```
Если папка есть, добавьте pyenv в PATH вручную (временно):
```
export PATH="$HOME/.pyenv/bin:$PATH"
```
Убедитесь, что pyenv работает:
```
pyenv --version
```
Отредактируйте ~/.bashrc в текстовом редакторе 
```
nano ~/.bashrc
```
Добавьте в конец файла строки:
```
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init - bash)"
```
Обновите конфигурацию:
```
source ~/.bashrc
```
Проверьте, что pyenv теперь доступен:
```
pyenv --version
```
Установите Python 3.9.16 и создайте venv:  
Скачайте архив Python 3.9.16:
```
cd /tmp
wget https://www.python.org/ftp/python/3.9.16/Python-3.9.16.tar.xz
```
Установите Python 3.9.16 через pyenv, используя скачанный архив  
Скопируйте архив в кэш pyenv:
```
mkdir -p ~/.pyenv/cache
cp /tmp/Python-3.9.16.tar.xz ~/.pyenv/cache/
```
Теперь установите Python:
```
pyenv install 3.9.16
```
После успешной установки проверьте:
```
pyenv versions
```
Вы должны увидеть:
```
* system (set by /home/user/.pyenv/version)
3.9.16
```
Создайте папку проекта и виртуальное окружение:
```
mkdir ~/rpicoral && cd ~/rpicoral
pyenv local 3.9.16          # фиксируем версию для этой папки
python -m venv venv
source venv/bin/activate
```
Проверьте версию Python:
```
python --version   # должно быть 3.9.16
```
Установка PyCoral из официального репозитория Google
```
pip install --upgrade pip
pip install --extra-index-url https://google-coral.github.io/py-repo/rpicoral ~=2.0
```
Даунгрейд NumPy в вашем виртуальном окружении:
```
pip install "numpy<2" #Должно быть 1.24.x или 1.26.x.
```
Проверка работы Coral TPU:
```
python -c "from pycoral.utils import edgetpu; print(edgetpu.list_edge_tpus())"
```
Вывод [{'type': 'pci', 'path': '/dev/apex_0'}] подтверждает, что устройство корректно инициализировано и доступно через драйвер.










