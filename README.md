# Tutorial de instalação do PDK IHP 130 nm e ferramentas Digitais no Ubuntu 24.04
**Autor:** Prof. Thiago Brito

Todos os comandos abaixo devem ser executados no **PowerShell** do Windows.

## 1. Instalação da WSL e distribuição Ubuntu
Para verificar a lista de versões do linux disponível, pode usar o seguinte comando:

```powershell
wsl --list --online
```

Escolher a versão:

```powershell
wsl --install -d "Ubuntu-24.04"
```

Depois atualizar os repositórios com o comando:

```bash
sudo apt-get update -q && sudo apt-get upgrade -q -y

exit
```

## 2. Instalação das ferramentas

Entrando novamente no ambiente pelo powershell

```powershell
wsl ~ -d "Ubuntu-24.04"
```

Agora podemos instalar o gedit, o python e as dependências:

```bash
sudo apt install -y gedit build-essential flex bison libx11-dev libxpm-dev libxext-dev libxft-dev tcl-dev tk-dev autoconf libtool libxaw7-dev libreadline-dev xterm libqt5designer5 libqt5multimedia5 libqt5opengl5t64 libqt5multimediawidgets5 libqt5printsupport5t64 libqt5sql5t64 libqt5xmlpatterns5 ruby ruby-dev libgit2-dev python3-venv python3-tk vim-gtk3 curl iverilog gtkwave
```

Preparando o ambiente Python virtual para ferramentas:

```bash
echo 'export PATH=$HOME/.local/bin:$PATH' >> .bashrc

python3 -m venv $HOME/.venv

echo "# Ambiente virtual Python ativado por padrão" >> ~/.bashrc
echo "# para desativar, executar comando 'deactivate'" >> ~/.bashrc
echo "if [ -f "$HOME/.venv/bin/activate" ]; then" >> ~/.bashrc
echo "source "$HOME/.venv/bin/activate"" >> ~/.bashrc
echo "fi" >> ~/.bashrc

source ~/.bashrc

pip install --upgrade pip

pip install psutil

pip install matplotlib

pip install numpy
```


Instalação e configuração do PDK IHP 130 nm

```bash
mkdir ~/cad

cd ~/cad

git clone --recursive https://github.com/IHP-GmbH/IHP-Open-PDK.git

cd IHP-Open-PDK

git checkout dev

echo "# Configuração do IHP-Open-PDK" >> ~/.bashrc
echo "export PDK_ROOT=\$HOME/cad/IHP-Open-PDK" >> ~/.bashrc
echo "export PDK=ihp-sg13g2" >> ~/.bashrc
echo "export KLAYOUT_PATH=\"\$HOME/.klayout:\$PDK_ROOT/\$PDK/libs.tech/klayout\"" >> ~/.bashrc
echo "export KLAYOUT_HOME=\$HOME/.klayout" >> ~/.bashrc
source ~/.bashrc
```

Instalar o klayout:

```bash
cd ~/cad

wget https://www.klayout.org/downloads/Ubuntu-24/klayout_0.30.3-1_amd64.deb

sudo dpkg -i klayout_0.30.3-1_amd64.deb

rm klayout_0.30.3-1_amd64.deb

klayout -e &
```

Instalar o nix para o funcionamento do sintetizador Librelane:

```bash
curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s --install --no-confirm --extra-conf "extra-substituters=https://nix-cache.fossi-foundation.org extra-trusted-public-keys=nix-cache.fossi-foundation.org:3+K59iFwXqKsL7BNu6Guy0v+uTlwsxYQxjspXzqLYQs="

exit
```

Com isso a primeira etapa de configuração foi concluída, agora precisamos instalar o librelane:

```bash
wsl ~ -d "Ubuntu-24.04"

cd ~/cad

git clone https://github.com/librelane/librelane

cd librelane

git checkout dev

##Nesse passo demora bastante
nix-shell 

exit

```

## 3. Execução de exemplo
Com tudo configurado, já podemos executar o exemplo com um somador completo:

```bash
mkdir ~/projetos/halfadd/

cd ~/projetos/halfadd

mkdir src

cd src

gedit halfadd.v
```

Copiar o código abaixo no editor de texto:

```bash
module halfadd( 	input a, b, 
output s, cout
);
assign s = a ^ b; assign cout = a & b;
endmodule
```

Agora salvamos o arquivo e vamos criar outro:

```bash
gedit halfadd_tb.v
```

Copiar o código abaixo no editor de texto:
```bash
module halfadd_tb;
reg a,b;
wire sum,carry;

halfadd uut(a,b,sum,carry);

initial begin

$dumpfile("halfadd_tb.vcd");
$dumpvars(0, halfadd_tb);
a = 0; b = 0; 
#10
a = 0; b = 1;
#10
a = 1; b = 0;
#10
a = 1; b = 1;
#10
#10
$finish();
end
                
endmodule
```

Agora vamos verificar o funcionamento:

```bash
iverilog -o fulladd_tb fulladd_tb.v fulladd.v halfadd.v 

vvp fulladd_tb

gtkwave fulladd_tb.vcd
```

Vai abrir a forma de onda e poderemos analisar o funcionamento do nosso circuito.

Com tudo funcionando corretamente. Agora salvamos o arquivo e vamos ajustar o arquivo de configuração:

```bash
cd ~/projetos/halfadd/

gedit config.json

{
    "DESIGN_NAME": "halfadd",
    "VERILOG_FILES": "dir::src/*.v",
    "CLOCK_PORT": "clk",
    "CLOCK_NET": "ref::$CLOCK_PORT",
    "FP_PIN_ORDER_CFG": "dir::pin_order.cfg",
    "FP_CORE_UTIL" : 10,
    "CLOCK_PERIOD": 10.0,
    "DESIGN_IS_CORE": true
}
```

Agora salvamos o arquivo e vamos ajustar o arquivo de pinagem:

```bash
cd ~/projetos/halfadd/

gedit pin_order.cfg

#N
a
b

#S
sum
cout

#W

#E
```

Agora salvamos o arquivo e sintetizar:

```bash
cd ~/cad/librelane

nix-shell

cd ~/projetos/halfadd

librelane --manual-pdk config.json

exit

```

Executando o klayout para vermos o gds:

```bash
klayout ~/projetos/halfadd/runs/RUN_*/58-klayout-streamout/gcd.klayout.gds -nn $PDK_ROOT/$PDK/libs.tech/klayout/tech/sg13g2.lyt -l $PDK_ROOT/$PDK/libs.tech/klayout/tech/sg13g2.lyp

```
