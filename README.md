Tutorial de instalação UNIC-CASS

Para verificar a lista de versões do linux disponível, pode usar o seguinte comando:

wsl --list --online

Escolher a versão:

wsl --install -d “Ubuntu-24.04”


Depois atualizar os repositórios com o comando:

sudo apt-get update -q && sudo apt-get upgrade -q -y


Aí podemos instalar o gedit e o python:

sudo apt install -y gedit build-essential flex bison libx11-dev \
libxpm-dev libxext-dev libxft-dev tcl-dev tk-dev autoconf libtool \
libxaw7-dev libreadline-dev xterm libqt5designer5 libqt5multimedia5 \
libqt5opengl5t64 libqt5multimediawidgets5 libqt5printsupport5t64 \
libqt5sql5t64 libqt5xmlpatterns5 ruby ruby-dev libgit2-dev python3-venv \
python3-tk vim-gtk3 curl

Preparando o ambiente Python virtual para ferramentas:

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

Instalar o klayout:

cd ~/cad

wget https://www.klayout.org/downloads/Ubuntu-24/klayout_0.30.3-1_amd64.deb

sudo dpkg -i klayout_0.30.3-1_amd64.deb

rm klayout_0.30.3-1_amd64.deb


wget https://www.klayout.org/downloads/Ubuntu-22/klayout_0.29.4-1_amd64.deb
klayout -e &


Instalação e configuração do PDK IHP 130 nm

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

cd $KLAYOUT_PATH/tech/drc/

#wget https://github.com/IHP-GmbH/IHP-Open-PDK/blob/dev/ihp-sg13g2/libs.tech/klayout/tech/drc/rule_decks/sg13g2_maximal.drc

#mv sg13g2_maximal.drc sg13g2_maximal.lydrc

pip install -r ../../../../../requirements.txt




Instalar o nix para o funcionamento:

curl --proto '=https' --tlsv1.2 -sSf -L https://install.determinate.systems/nix | sh -s --install --no-confirm --extra-conf "extra-substituters=https://nix-cache.fossi-foundation.org extra-trusted-public-keys=nix-cache.fossi-foundation.org:3+K59iFwXqKsL7BNu6Guy0v+uTlwsxYQxjspXzqLYQs="

exit

Com isso a primeira etapa de configuração foi concluída, agora precisamos instalar o librelane:

wsl ~ -d "Ubuntu-24.04"

mkdir ~/cad

cd ~/cad

git clone https://github.com/librelane/librelane

cd librelane

git checkout dev

nix-shell

#librelane --manual-pdk --smoke-test

Com tudo configurado, já podemos executar o exemplo:

cd ~/

mkdir ~/gcd_example

cd ~/cad/librelane/

nix-shell

cd ~/gcd_example

wget https://unic-cass.github.io/training/files/gcd.tar.gz

tar xvf gcd.tar.gz

rm gcd.tar.gz

Com os arquivos do exemplos baixados, podemos editar os arquivos:

gedit config.json

{
	"DESIGN_NAME": "gcd",
	"VERILOG_FILES": "dir::src/*.v",
	"CLOCK_PORT": "clk",
	"CLOCK_NET": "ref::$CLOCK_PORT",
	"FP_PIN_ORDER_CFG": "dir::pin_order.cfg",
	"CLOCK_PERIOD": 10.0,
	"DESIGN_IS_CORE": true
}


gedit pin_order.cfg

#N
clk
reset
operands_val
#S
operands_bits_B.*
#W
operands_bits_A.*
operands_rdy
result_rdy
#E
result_bits_data.*
result_val

Depois das modificações nos arquivos, podemos executar o fluxo digital com o librelane:

flow.tcl -design gcd -ignore_mismatches

exit


Executando o klayout para vermos o gds:


klayout openlane/gcd/runs/RUN_*/results/signoff/gcd.gds -nn $PDK_ROOT/$PDK/libs.tech/klayout/tech/$PDK.lyt -l $PDK_ROOT/$PDK/libs.tech/klayout/tech/$PDK.lyp

Agora vamos fazer um projeto usando o Icarus e Gtkwave:

sudo apt-get install iverilog gtkwave

Exemplo com um somador completo:

mkdir fulladd

cd fulladd

docker run -it -v $(pwd):/fulladd/ -v $PDK_ROOT:$PDK_ROOT -e PDK_ROOT=$PDK_ROOT -u $(id -u $USER):$(id -g $USER) efabless/openlane:latest

cd /fulladd

flow.tcl -design fulladd -init_design_config

cd /fulladd/openlane/fulladd

exit

mkdir openlane/fulladd/src

cd openlane/fulladd/src

gedit fulladd.v



Copiar o código abaixo no editor de texto:


module fulladd ( 	input a, b, cin, 
output sum, cout 
 );
wire s1, c1, c2;
halfadd u1(a, b, s1, c1); 
halfadd u2(s1, cin, sum, c2); 
assign cout = c1 | c2;
endmodule

module halfadd( 	input a, b, 
output s, cout
);
assign s = a ^ b; assign cout = a & b;
endmodule



Agora salvamos o arquivo e vamos criar outro:

gedit fulladd_tb.v

module fulladd_tb;
reg a,b,cin;
wire sum,carry;

fulladd uut(a,b,cin,sum,carry);

initial begin

$dumpfile("fulladd_tb.vcd");
$dumpvars(0, fulladd_tb);
a = 0; b = 0; cin = 0;
#10
a = 0; b = 0; cin = 1;
#10
a = 0; b = 1; cin = 0;
#10
a = 0; b = 1; cin = 1;
#10
a = 1; b = 0; cin = 0;
#10
a = 1; b = 0; cin = 1;
#10
a = 1; b = 1; cin = 0;
#10
a = 1; b = 1; cin = 1;
#10
$finish();
end
                
endmodule


Agora vamos verificar o funcionamento:

iverilog -o fulladd_tb fulladd_tb.v fulladd.v halfadd.v 

vvp fulladd_tb

gtkwave fulladd_tb.vcd

Vai abrir a forma de onda e poderemos analisar o funcionamento do nosso circuito:



Com tudo funcionando corretamente. Agora salvamos o arquivo e vamos ajustar o arquivo de configuração:

cd ~/fulladd/openlane/fulladd/

gedit config.json

{
    "DESIGN_NAME": "fulladd",
    "VERILOG_FILES": "dir::src/*.v",
    "CLOCK_PORT": "clk",
    "CLOCK_NET": "ref::$CLOCK_PORT",
    "FP_PIN_ORDER_CFG": "dir::pin_order.cfg",
    "FP_CORE_UTIL" : 10,
    "CLOCK_PERIOD": 10.0,
    "DESIGN_IS_CORE": true
}

Agora salvamos o arquivo e vamos ajustar o arquivo de pinagem:

cd ~/fulladd/openlane/fulladd/

gedit pin_order.cfg

#N
a
b
cin
#S
sum
cout
#W

#E

Agora salvamos o arquivo e sintetizar:


cd ~/fulladd 

docker run -it -v $(pwd):/fulladd -v $PDK_ROOT:$PDK_ROOT -e PDK=$PDK -e PDK_ROOT=$PDK_ROOT -u $(id -u $USER):$(id -g $USER) efabless/openlane:latest

cd /fulladd/openlane

flow.tcl -design fulladd -ignore_mismatches

exit






Executando o klayout para vermos o gds:

klayout openlane/fulladd/runs/RUN_*/results/signoff/fulladd.gds -nn $PDK_ROOT/$PDK/libs.tech/klayout/tech/$PDK.lyt -l $PDK_ROOT/$PDK/libs.tech/klayout/tech/$PDK.lyp


Focar no openlane para o treinamento. Essa parte de baixo não é necessária por enquanto.



Nessa parte é para integrar o projeto ao caravel, primeiro vamos fazer alguns exemplos.

Agora iremos configurar o caravel, primeiro instalando mais um pacote do python:

sudo apt-get install python3-venv -y

Vamos fazer o download do código fonte:

git clone -b mpw-9k https://github.com/efabless/caravel_user_project uniccass_example

cd uniccass_example

Agora vamos compilar::

make setup


