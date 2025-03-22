# Windows Server Active Directory


## Objectivo do Projecto


### Competências Adquiridas

## Passo a Passo

### Criação da Maquina virtual
 <br/>
    <br/>
Vamos começar com a criação de uma máquina virtual utilizando o Virtualbox. 
Para adicionar uma máquina nova a partir de uma ISO previamente descarregada, clicamos em "New".  


<p align="center">
  <br/>
  Machine -> New
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/371b4728-a7cc-48fd-9b1b-72ab5378b7dd" height="80%" width="80%"/>
    <br/>
    <br/>
  <p/>
    
De seguida damos o nome a máquina, escolhemos a pasta para onde serão guardados todos os ficheiros que são criados durante a criação da máquina, tal como o disco virtual, e indicamos o ficheiro ISO que vamos usar para a criação. Clique `Next`  


**Nota:** Selecionei a opção `Skip unattended installation` porque é do meu interesse proceder a instalação manual do sistema operativo. 
<br/>
    <br/>

<p align="center">
  <br/>
  Seleção do Sistema Operativo
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/1fc6b161-515b-43e1-b585-5c50e073c211" height="60%" width="60%"/>
<p/> 
  <br/>
  <br/>

Nas janelas seguintes escolhi as especificações de Hardware da máquina virtual. Para este projeto **50GB** de disco, **4GB** de RAM e **4** CPUs são suficientes.  


<p align="center">
  <br/>
  Tamanho do Disco
  <br/>
  <br/>
  <img src="https://github.com/user-attachments/assets/f3f9b919-3dcd-41b2-8bbc-60e28c61e927" height="60%" width="60%"/>
  <br/>
  <br/>
<p/> 
  
  Na janela de **Hardware** Selecionei a opção **EFI**, mais em baixo explicarei esta opção bem como Secure Boot que irei selecionar e porquê.
  <br/>
  <br/>
<p align="center">
  Hardware
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/c3cadb82-1964-46eb-acf7-0b593280ff53" height="60%" width="60%"/>
  <br/><br/>
  Resumo final das opções selecionadas.
  <br/><br/>
  <img src="https://github.com/user-attachments/assets/cefccaaa-a482-4882-b614-8a3884f5ee10" height="60%" width="60%"/>
<p/>  
<br/>
<br/>
  
### Settings
Antes de Iniciar a máquina e proceder a instalação do Windows Server 2019 a partir da ISO previamente montada, existem algumas opções e configurações que irei selecionar.  
É possível aceder as configurações da maquina clicando em `Settings`com a máquina selecionada. 
<p align="center">
  <br/>
  <img src="https://github.com/user-attachments/assets/2886d5c6-f69b-4b6f-be79-d0985594675b" height="60%" width="60%"/>
<p/> 
  <br/> <br/>
 Dentro do Menu `Settings` activei duas opções que irão aumentar a performance e a segurança da maquina. 

 **EFI (Extensible Firmware Interface):** É uma interface de firmware que substitui a BIOS tradicional, oferecendo um boot mais rápido, suporte para discos grandes (GPT) e recursos avançados de segurança e compatibilidade.

**Secure Boot:** É uma funcionalidade do EFI/UEFI que impede a execução de software não assinado durante a inicialização do sistema, ajudando a proteger contra malware e rootkits ao garantir que apenas código confiável seja carregado.
 
 <br/> <br/>
 <p align="center">
  <img src="https://github.com/user-attachments/assets/e6696933-611d-4390-8009-c886d3f96af8" height="60%" width="60%"/>
  <br/> <br/>
 <p/> 
 Na aba processador, ativei a opção `PAE/NX`
  
 **PAE (Physical Address Extension):** Permite que processadores de 32 bits acessem mais de 4 GB de RAM, estendendo o endereçamento para até 64 GB. 

**O NX (No eXecute)** protege contra ataques que tentam executar código malicioso em áreas de memória que deveriam conter apenas dados. Os principais tipos de ataques que ele mitiga incluem:
- **Buffer Overflow com Execução de Código** – Um atacante pode tentar sobrescrever áreas da memória com código malicioso e executá-lo. O NX impede essa execução em regiões de dados, como a pilha (stack) e o monte (heap).

- **Return-to-libc** – Em vez de injetar código, o atacante tenta reutilizar funções legítimas da biblioteca C para executar comandos maliciosos. Embora o NX não impeça diretamente esse ataque, ele força os atacantes a usar técnicas mais avançadas, como ROP (Return-Oriented Programming).

- **Ataques de Shellcode** – O NX evita que código arbitrário injetado na memória seja executado, dificultando a execução de shellcodes maliciosos.
<br/> <br/>


<p align="center">  
  <br/> <br/>
  <img src="https://github.com/user-attachments/assets/857746a6-d3b0-4ce5-bbc9-8cf7a41933d1" height="60%" width="60%"/>
<p/>
 <br/> <br/>
 
 #### Configuração de placa de rede

 De seguida configurei dois adaptadores de rede, 1 ligado a rede externa, em NAT, e outro ligado a rede interna. 
 <br/> <br/>
 <p align="center"> 
  <img src="https://github.com/user-attachments/assets/0033b34b-f3dd-4085-8ef8-1b1cc047768a" height="60%" width="60%"/>
   <br/> <br/>
  <img src="https://github.com/user-attachments/assets/fa3c3a84-4bf2-4982-bf92-65237cf9ba0a" height="60%" width="60%"/>
<p/> 
<br/>
<br/>
  
### Iniciar a Maquina e instalação do SO  
De seguida iniciamos a máquina, procedendo para a instalação do sistema operativo. Basta seguir os passos que mostra nas imagens.  
    <br/> <br/>
<p align="center">
  <img src="https://github.com/user-attachments/assets/9dbb0776-27b6-4a15-8d3d-f544cf8371c7" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/ec85779a-212d-4894-8c35-cdea23cf0acb" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/c0d27ba3-4884-4dae-92d1-a32a18909c2c" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/c5f56a00-a933-4768-877f-c443914d3289" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/3f64bd6e-5b4b-46ad-aedd-bb8fe6dfe09e" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/510c6116-9d86-4bdc-af6c-1b6a49605afc" height="50%" width="50%"/>
<p/> 
<br/>
<br/>
  
### Configuração do Servidor

Primeira coisa a fazer é mudar o nome do Servidor, e atribuir um sufixo DNS. No meu caso dei nome ao Servidor Marcio e o sufixo DNS pilao.pt que irá ser o nome da minha rede.
<br/>
<p align="center">  
  <img src="https://github.com/user-attachments/assets/72fe12bf-4eea-4d43-ab64-dc7b2c7fe1dd" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/9028201a-3669-43c1-8c15-f86ddcb06c56" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/2a4584ec-7609-4f2a-8842-08f767db9b1e" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/8bd19136-3846-46d5-bacb-1327d4605fce" height="50%" width="50%"/>
 <p/>
<br/>
 <br/>
 De seguida é necessário reiniciar a máquina para as alterações serem aplicadas. 
 <br/>
 <br/>
 <p align="center"> 
  <img src="https://github.com/user-attachments/assets/652dcc53-b215-47f3-8d47-119c6ee73c19" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/de2832ad-4dea-4518-b413-da5ea09a319c" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/d334ac62-5249-4754-882e-7fd4335abfa9" height="50%" width="50%"/>
  <img src="https://github.com/user-attachments/assets/8683d05b-cd24-4288-ab65-dfc0afb72a0c" height="50%" width="50%"/>
 
  


 
 

  
<p/>

