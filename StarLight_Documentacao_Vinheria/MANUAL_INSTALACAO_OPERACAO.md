# Manual de instalação e operação | StarLight

**Projeto:** Monitoramento de Vinheria, CP5 Vinheria Full  
**Disciplina:** Edge Computing Systems, FIAP  
**Professor:** Prof. Dr. Fábio Henrique Cabrini  
**Revisão:** 30 de outubro de 2026

Este manual orienta a montagem do protótipo, a preparação do FIWARE, a execução do dashboard e os testes de integração. Os caminhos e GPIOs correspondem à cópia local examinada em `Site-Vinheria-revisado`.

O sistema mede temperatura, umidade e luminosidade relativa. A ESP32 publica por MQTT, o IoT Agent atualiza o Orion, uma assinatura encaminha dados ao STH-Comet e o dashboard consulta as APIs. O RFID identifica a credencial que reconheceu um alerta.

**Leitura importante:** há diferenças entre a implementação examinada e o enunciado do CP5. Os alarmes ambientais são locais ao firmware; o controle remoto existente é manual. Os triggers do dashboard não são enviados à ESP32. Não foi identificado LED azul independente piscando automaticamente por comando do front-end. O README contém a matriz desses requisitos.

## 1. Materiais e pré-requisitos

### Computador e software

| Recurso | Utilização |
|---|---|
| Computador com navegador | Dashboard, Wokwi e configuração |
| Git | Obter os repositórios |
| Node.js compatível com Vite 8 | Executar o dashboard; usar Node 22.12 ou versão superior compatível |
| npm | Instalar as dependências JavaScript |
| Arduino IDE | Compilar e gravar o dispositivo físico |
| Pacote ESP32 da Espressif | Suporte à placa no Arduino |
| Cabo USB de dados | Alimentação e gravação da ESP32 |
| Driver USB correspondente à placa | Reconhecimento da porta, se necessário |
| Docker e Docker Compose | Executar FIWARE na máquina de infraestrutura |
| Acesso à EC2 e SSH | Administrar a infraestrutura AWS utilizada pela equipe |

Python e pip não são necessários para executar esta versão do StarLight. Não existe procedimento FastAPI a seguir no código examinado.

### Hardware

- ESP32 DevKit.
- DHT11 para a montagem física.
- LDR com divisor resistivo ou módulo com saída analógica.
- LED RGB, configurado como cátodo comum.
- Três resistores limitadores de corrente para o RGB.
- Buzzer passivo compatível com o circuito de acionamento.
- LCD 16x2 com interface I2C, configurado em 0x27.
- Leitor RFID RC522 e cartões ou tags.
- Protoboard, jumpers, alimentação USB e suportes para fixação.
- Resistência de pull-up para DHT sem módulo, quando necessária.
- Conversão de nível I2C se o backpack do LCD utilizar pull-ups de 5 V.
- Encapsulamento inspirado em uma garrafa, conforme o projeto da equipe.

A simulação usa DHT22. Não substitua o DHT11 físico por DHT22 sem alterar o tipo configurado no firmware.

### Rede

O ESP32 clássico utiliza Wi-Fi de 2,4 GHz. A rede deve permitir alcançar o broker MQTT. Redes com portal de autenticação podem impedir a conexão do firmware.

O computador que executa o proxy precisa alcançar Orion, IoT Agent e STH-Comet. A conectividade HTTP do dashboard não comprova, por si só, o acesso MQTT da placa.

## 2. Obter e organizar os arquivos

Repositórios:

- StarLight: https://github.com/EstelaMariano/Vinheria-Full
- Infraestrutura: https://github.com/fabiocabrini/fiware
- Wokwi: https://wokwi.com/projects/479153212294129665

Quando os fontes estiverem disponíveis no repositório de entrega:

```bash
git clone https://github.com/EstelaMariano/Vinheria-Full.git
cd Vinheria-Full
```

Localize a pasta que contém `package.json`. Na cópia local revisada, ela se chama `Site-Vinheria-revisado`. Os comandos npm deste manual devem ser executados nessa pasta.

O comando informado pela equipe:

```bash
git remote add origin https://github.com/EstelaMariano/Vinheria-Full.git
```

apenas associa um repositório Git local a um endereço remoto. Não instala dependências, não envia os arquivos e não inicia o sistema. Antes de usá-lo em um repositório existente, confira `git remote -v`; se `origin` já existir, revise seu destino em vez de executar `add` novamente.

Para publicar esta documentação, mantenha os documentos na mesma pasta e preserve:

```text
README.md
MANUAL_INSTALACAO_OPERACAO.md
docs/images/arquitetura-starlight.png
```

A imagem é um arquivo de apoio necessário para a renderização do README no GitHub. O pacote de entrega já contém essa organização.

Na cópia local revisada, o `.gitignore` ignora `README.md` e `docs/`. Antes da publicação, remova essas regras para os documentos finais ou adicione explicitamente os arquivos desejados com `git add -f README.md MANUAL_INSTALACAO_OPERACAO.md docs/images/arquitetura-starlight.png`. Confira o conteúdo preparado com `git diff --cached --stat` antes de criar o commit. Publicar apenas o Markdown, sem a imagem, deixa a ilustração indisponível no GitHub.

## 3. Montagem física

Desconecte a alimentação antes de alterar conexões. Identifique os pinos pelo nome impresso na placa e no módulo, não apenas pela posição visual.

Todos os módulos devem compartilhar GND. Os sinais conectados ao ESP32 devem respeitar 3,3 V. Não aplique 5 V nos GPIOs ou na entrada ADC.

### Pinagem do perfil físico

| Componente | Pino do componente | ESP32 | Alimentação e observação |
|---|---|---|---|
| DHT11 | DATA | GPIO 4 | Alimentar de modo compatível com sinal de 3,3 V |
| DHT11 | VCC e GND | 3V3 e GND | Conferir pinagem do módulo |
| LDR ou módulo analógico | AO ou ponto central do divisor | GPIO 34 | Tensão entre 0 e 3,3 V |
| LDR, se módulo | VCC e GND | 3V3 e GND | Saída DO não é utilizada |
| RGB | R | GPIO 25 | Resistor em série |
| RGB | G | GPIO 26 | Resistor em série |
| RGB | B | GPIO 27 | Resistor em série |
| RGB cátodo comum | COM | GND | `RGB_COMMON_ANODE = false` |
| Buzzer passivo | Entrada de acionamento | GPIO 13 | Usar estágio de acionamento se a corrente exigir |
| Buzzer | Retorno | GND | Conferir o componente físico |
| LCD I2C | SDA | GPIO 21 | Nível lógico de 3,3 V no ESP32 |
| LCD I2C | SCL | GPIO 22 | Nível lógico de 3,3 V no ESP32 |
| LCD I2C | VCC e GND | Fonte compatível e GND | Avaliar pull-ups do backpack |
| RC522 | 3.3V e GND | 3V3 e GND | Não alimentar com 5 V |
| RC522 | SDA/SS | GPIO 5 | Seleção SPI, não SDA do barramento I2C |
| RC522 | SCK | GPIO 18 | Clock SPI |
| RC522 | MISO | GPIO 19 | SPI |
| RC522 | MOSI | GPIO 23 | SPI |
| RC522 | RST | GPIO 33 | Reset |
| RC522 | IRQ | Sem conexão | Não utilizado pelo sketch |

O pino DHT com rótulo `SDA` no Wokwi é a linha de dados desse sensor; isso não significa que ele deva ser ligado ao SDA do LCD. Um pino `NC` de sensor avulso deve permanecer desconectado.

### DHT11

1. Confira se está usando sensor avulso ou módulo com três terminais.
2. Ligue VCC e GND conforme a identificação do fabricante.
3. Conecte DATA ao GPIO 4.
4. Para sensor avulso, utilize pull-up de DATA para 3,3 V conforme o datasheet, tipicamente entre 4,7 kΩ e 10 kΩ. Módulos podem já incorporar esse resistor.
5. Mantenha o sensor ventilado e afastado do calor do regulador da ESP32.

Não use a pinagem antiga de testes que empregava GPIO 2.

### LDR

O GPIO 34 é somente entrada e recebe o sinal analógico. Para um módulo, conecte AO ao GPIO 34, VCC a 3,3 V e GND ao terra comum.

Se utilizar LDR avulso, monte um divisor resistivo. Para acompanhar a direção assumida pelo firmware, use o resistor fixo entre 3,3 V e o ponto de leitura, e o LDR entre esse ponto e GND. Um resistor de aproximadamente 10 kΩ pode servir como ponto inicial de bancada; o valor final depende do LDR e da faixa desejada.

O firmware calcula a média de 12 amostras e converte ADC de 0 a 4095 em percentual de 100 a 0. Nesse arranjo, maior luz reduz a leitura analógica e aumenta o percentual publicado. Cubra e ilumine o sensor para confirmar o sentido antes de validar alertas.

O percentual é relativo à eletrônica e ao mapeamento utilizado. Não o apresente como lux.

### LED RGB

1. Identifique os canais R, G, B e o terminal comum.
2. Coloque um resistor em série com cada canal. Valores entre 220 Ω e 330 Ω são referências usuais de bancada, mas devem ser verificados para o LED e a corrente escolhidos.
3. Ligue R ao GPIO 25, G ao GPIO 26 e B ao GPIO 27.
4. Ligue o comum ao GND no modelo de cátodo comum.
5. Se o LED for ânodo comum, revise alimentação e ajuste `RGB_COMMON_ANODE` no código.

O circuito simulado local liga os canais diretamente. Essa simplificação da simulação não dispensa resistores na montagem real.

O RGB não substitui automaticamente o requisito de LED azul do ESP32 piscando remotamente. Esse mecanismo obrigatório não foi identificado no firmware examinado.

### Buzzer

Use buzzer passivo para reproduzir frequências diferentes. O sinal de controle físico está no GPIO 13. Verifique corrente e tensão do componente; se não puder ser acionado pelo GPIO, utilize transistor ou driver dimensionado.

O buzzer do Wokwi utiliza outro GPIO. Não copie sua ligação do GPIO 18 para a montagem física, porque nesse perfil o GPIO 18 é o clock do RC522.

### LCD I2C

O código utiliza `LiquidCrystal_I2C lcd(0x27, 16, 2)`. Conecte SDA ao GPIO 21 e SCL ao GPIO 22.

Muitos backpacks são alimentados em 5 V e possuem resistores de pull-up ligados a essa tensão. Nesses casos, use conversor bidirecional de nível entre o ESP32 e o barramento do LCD. Não suponha que alimentar o módulo em 5 V torna seus sinais seguros para o ESP32.

Se não aparecer texto, ajuste o contraste do módulo e confira endereço, alimentação e conexões. Um endereço diferente exige alteração de `LCD_ADDRESS`.

### RFID RC522

Ligue o leitor à interface SPI indicada na tabela. O terminal SDA do RC522 funciona como SS nesse uso. Alimente o módulo com 3,3 V.

Mantenha os fios curtos e confira SS, RST e clock antes de testar. Aproxime a tag da antena do módulo. Não instale a antena atrás de metal que prejudique a leitura.

## 4. Encapsulamento

A identidade visual da garrafa aproxima o dispositivo do ambiente da vinheria. O material consultado não estabelece dimensões, material construtivo ou posições internas definitivas, portanto estes dados não são presumidos.

Na montagem:

1. Fixe a placa e os módulos sem deixar terminais encostarem entre si.
2. Garanta ventilação para o DHT.
3. Mantenha uma abertura para o LDR observar a luz externa.
4. Posicione o buzzer de modo que o som seja audível.
5. Deixe LCD e sinalização visíveis.
6. Deixe a região de aproximação do RFID acessível.
7. Preserve acesso ao cabo USB e espaço para manutenção.
8. Compare as leituras antes e depois de fechar a estrutura, verificando influência de calor e sombra.

Fotografe o exterior e a montagem interna para a avaliação do encapsulamento.

## 5. Instalar e configurar a Arduino IDE

1. Instale a Arduino IDE a partir do site oficial.
2. Nas preferências, se o ESP32 ainda não estiver disponível, adicione a URL de placas da Espressif:

```text
https://espressif.github.io/arduino-esp32/package_esp32_index.json
```

3. Abra o Gerenciador de Placas e instale **esp32 by Espressif Systems**.
4. Conecte a placa com um cabo USB de dados.
5. Selecione **ESP32 Dev Module**, verificando se corresponde à sua placa.
6. Selecione a porta serial correspondente.
7. Instale as bibliotecas pelo Gerenciador de Bibliotecas.

| Biblioteca | Cabeçalho | Uso |
|---|---|---|
| PubSubClient, Nick O'Leary | `PubSubClient.h` | MQTT |
| DHT sensor library, Adafruit | `DHT.h` | DHT11 e DHT22 |
| Adafruit Unified Sensor | Dependência da biblioteca DHT | Suporte aos sensores |
| LiquidCrystal I2C, Frank de Brabander | `LiquidCrystal_I2C.h` | LCD |
| MFRC522, GithubCommunity | `MFRC522.h` | RFID |

`WiFi.h`, `Wire.h`, `SPI.h`, `Preferences.h` e `time.h` pertencem ao suporte da plataforma utilizado pelo firmware. Não instale versões aleatórias de bibliotecas homônimas para substituir componentes do core ESP32.

Abra:

```text
firmware/physical/starlight_physical/starlight_physical.ino
```

O nome da pasta deve acompanhar o nome do sketch. Mantenha `starlight_secrets.h` ao lado do `.ino`.

### Configurar Wi-Fi

No arquivo `starlight_secrets.h`, preencha:

```cpp
#define STARLIGHT_WIFI_SSID "SEU_WIFI"
#define STARLIGHT_WIFI_PASSWORD "SUA_SENHA"
```

Não publique esse arquivo com credenciais reais.

### Configurar o broker e gerar os perfis

Na raiz da aplicação:

```bash
npm run configure:host -- SEU_IPV4_PUBLICO
```

Esse comando grava o host e gera os firmwares. Para gerar novamente após editar o código-base:

```bash
npm run firmware:generate
```

O arquivo-base é `firmware/starlight_esp32/starlight_esp32.ino`. Alterações permanentes de comportamento, limites e tags devem ser feitas nele e depois propagadas pelo gerador. Editar apenas o sketch gerado pode fazer a alteração desaparecer na próxima geração.

Confira `STARLIGHT_MQTT_BROKER` no arquivo gerado. O host deve conter somente IP ou DNS, sem `http://`, porta ou caminho.

### Compilar e gravar

1. Clique em **Verificar**.
2. Corrija qualquer biblioteca ausente antes de continuar.
3. Clique em **Upload**.
4. Se a placa não entrar no modo de gravação, siga o procedimento BOOT/EN correspondente ao modelo.
5. Abra o Monitor Serial em **115200 baud**.
6. Confirme mensagens de conexão Wi-Fi e MQTT e linhas de leitura dos sensores.

Logs de conexão indicam etapas individuais; a confirmação completa exige que o dashboard receba dados do mesmo Device ID.

### Erro LiquidCrystal_I2C.h ausente

A mensagem:

```text
fatal error: LiquidCrystal_I2C.h: No such file or directory
```

significa que o compilador não encontrou o cabeçalho. Abra o Gerenciador de Bibliotecas, instale **LiquidCrystal I2C** compatível com a API `init()` e `backlight()` usada pelo sketch, e compile novamente.

Se já estiver instalada, verifique instalações duplicadas, a pasta de bibliotecas utilizada pela IDE e qual biblioteca aparece no log detalhado da compilação. Apenas criar um arquivo vazio com esse nome não resolve o problema.

## 6. Preparar a infraestrutura FIWARE

### AWS EC2

1. Abra o ambiente AWS Academy autorizado para a atividade.
2. Inicie o laboratório e a instância utilizada pela equipe.
3. Anote o **IPv4 público atual**.
4. Identifique a chave privada e o usuário SSH da imagem.
5. Em uma imagem Ubuntu, conecte usando o padrão:

```bash
ssh -i CAMINHO_DA_CHAVE.pem ubuntu@SEU_IPV4_PUBLICO
```

Substitua os marcadores pelos valores da instância. Não compartilhe a chave privada.

| Endereço | Significado |
|---|---|
| IPv4 público | Acesso do computador externo e do ESP32 à instância |
| IPv4 privado | Comunicação da VPC; não é normalmente alcançável pela internet |
| `localhost` | A própria máquina ou contêiner de onde o comando é executado |
| `fiware-orion` ou alias Docker | Nome resolvido na rede Docker em que foi definido |
| ID da instância | Identificador de administração AWS; não é endereço de broker |

Ao parar e iniciar a instância, o IPv4 público pode mudar. Atualize também o firmware e grave novamente a placa quando necessário.

### Docker e Compose

Na máquina de infraestrutura, confirme:

```bash
docker --version
docker compose version
```

Se faltarem, instale Docker e o plugin Compose pelo procedimento oficial compatível com a distribuição. O material do professor também utiliza o comando legado `docker-compose`; verifique qual está disponível.

Obtenha e inicie a pilha:

```bash
git clone https://github.com/fabiocabrini/fiware.git
cd fiware
docker compose up -d
docker compose ps
```

Se houver restrição de permissão sobre o socket Docker, utilize o procedimento administrativo da máquina, inclusive `sudo` quando apropriado.

A saída de `ps` deve listar os serviços sem reinicialização contínua. Verifique os logs:

```bash
docker compose logs --tail=100
```

Não remova volumes para corrigir um erro de conexão. Os volumes guardam dados; sua exclusão pode apagar entidades e históricos.

### Portas

| Serviço | Porta | Origem que precisa acessar |
|---|---:|---|
| Orion | 1026/TCP | Proxy da aplicação e componentes internos |
| IoT Agent, administração | 4041/TCP | Proxy e integrações de cadastro |
| IoT Agent, endpoint de dispositivo | 7896/TCP, se publicado | Apenas integrações que realmente o utilizem |
| STH-Comet | 8666/TCP | Proxy e Orion pela rede interna |
| Mosquitto | 1883/TCP | ESP32, Wokwi e IoT Agent |
| MongoDB | 27017 no mapeamento aplicável | Serviços internos; não precisa ficar público |
| Vite local | Normalmente 5173/TCP | Navegador local |
| SSH | 22/TCP | Administração autorizada |

Revise os mapeamentos efetivos com `docker compose ps`. Na AWS, libere no Security Group as origens necessárias. Para o Wokwi, a origem MQTT não é necessariamente o IP do computador. Se precisar ampliar temporariamente acesso para a demonstração, restrinja novamente depois.

Não exponha MongoDB diretamente para o dashboard. Os desenhos conceituais de portas não substituem o arquivo Compose executado.

### Validar cada API na EC2

```bash
curl -i http://localhost:1026/version
curl -i http://localhost:4041/iot/about
curl -i http://localhost:8666/version
```

Espere resposta HTTP válida e conteúdo de identificação do serviço. Uma falha deve ser investigada individualmente nos logs e mapeamentos.

Repita do computador usando o IPv4 público. No Windows PowerShell:

```powershell
curl.exe -i http://SEU_IPV4_PUBLICO:1026/version
curl.exe -i http://SEU_IPV4_PUBLICO:4041/iot/about
curl.exe -i http://SEU_IPV4_PUBLICO:8666/version
Test-NetConnection SEU_IPV4_PUBLICO -Port 1883
```

Sucesso TCP em 1883 indica alcance da porta, não autenticação MQTT nem recebimento de telemetria.

## 7. Executar o dashboard

Na pasta com `package.json`:

```bash
npm install
npm run dev
```

Abra o endereço fornecido pelo terminal. O Vite normalmente usa 5173, mas pode selecionar outra porta se ela estiver ocupada.

### Configuração local

A referência de variáveis está em `.env.example`. Se necessário, copie para `.env.local`:

```powershell
Copy-Item .env.example .env.local
```

Preencha `FIWARE_HOST` com o IPv4 público ou DNS alcançável. Mantenha service e service path consistentes:

```dotenv
FIWARE_HOST=SEU_IPV4_PUBLICO
FIWARE_REQUEST_TIMEOUT_MS=12000
FIWARE_SERVICE=smart
FIWARE_SERVICE_PATH=/
VITE_FIWARE_SERVICE=smart
VITE_FIWARE_SERVICE_PATH=/
VITE_FIWARE_API_KEY=TEF
VITE_IOT_CBROKER_URL=http://fiware-orion:1026
VITE_STH_NOTIFY_URL=http://fiware-sth-comet:8666/notify
VITE_ENTITY_ID=urn:ngsi-ld:WineCellar:garrafa011
VITE_ENTITY_TYPE=WineCellar
```

O arquivo admite overrides avançados `FIWARE_ORION_URL`, `FIWARE_IOT_URL` e `FIWARE_STH_URL`. Valores incorretos nesses overrides podem manter destinos antigos mesmo com `FIWARE_HOST` atualizado.

No uso comum, prefira **Dispositivos > Infraestrutura FIWARE > Salvar e testar**. Essa operação local atualiza os destinos do proxy e gera as variantes de firmware. A alteração de arquivos pelo terminal exige reiniciar o servidor de desenvolvimento.

Os nomes Docker em `VITE_IOT_CBROKER_URL` e `VITE_STH_NOTIFY_URL` são usados pela integração entre serviços. Confirme esses nomes na rede da pilha. Não os substitua indiscriminadamente pelo IP público.

### Versão hospedada

A configuração do ambiente de hospedagem é separada da configuração local. A pasta `worker/` e os scripts de build pertencem a essa versão:

```bash
npm run build:site
npm run validate:site
```

Um build estático isolado não fornece automaticamente as rotas `/api/*`. A hospedagem precisa executar a integração de servidor correspondente. O funcionamento da versão hospedada não foi confirmado nesta revisão.

## 8. Provisionar os dispositivos

Abra **Dispositivos**, salve o host e aguarde o resultado dos testes. Provisione primeiro o perfil que será utilizado.

| Perfil | Device ID | Entity ID |
|---|---|---|
| Simulação Wokwi | `wokwi001` | `urn:ngsi-ld:WineCellar:wokwi001` |
| Adega física | `garrafa011` | `urn:ngsi-ld:WineCellar:garrafa011` |

1. Selecione o perfil.
2. Clique em **Provisionar**.
3. Para um cadastro existente, use **Atualizar provisionamento**.
4. Aguarde a confirmação.
5. Ligue a placa ou inicie a simulação com o mesmo Device ID.

Para cadastrar outra unidade, preencha perfil, nome e Device ID. O nome é uma identificação visual; o Device ID participa dos tópicos e deve coincidir com o firmware. O sistema gera uma URN e permite ajustes avançados de entidade e tipo.

### O que o provisionamento faz

- **Grupo de serviço:** configura parâmetros compartilhados da integração, incluindo API key e destino do broker de contexto.
- **Dispositivo no IoT Agent:** associa Device ID, transporte MQTT, atributos e comandos à entidade.
- **Entidade Orion:** representa o dispositivo e seus valores atuais.
- **Assinatura histórica:** encaminha atualizações ao STH-Comet.

Service `smart` e service path `/` devem acompanhar cadastro e consultas. Uma entidade em outro namespace pode não aparecer nas consultas atuais.

A assinatura utiliza destino interno como `http://fiware-sth-comet:8666/notify`, formato `legacy` e metadados com `TimeInstant`. Esse destino é acessado pelo Orion, não pelo navegador.

### Contrato MQTT e comandos

| Uso | Padrão |
|---|---|
| Telemetria | `/TEF/DEVICE_ID/attrs` |
| Comando | `/TEF/DEVICE_ID/cmd` |
| Resultado | `/TEF/DEVICE_ID/cmdexe` |

O payload de telemetria usa UltraLight 2.0, com pares separados por `|`. Exemplo ilustrativo coerente com os object IDs:

```text
t|13.5|h|70|l|35
```

| Comando | Valores aceitos |
|---|---|
| `setAlertMode` | `on`, `off` |
| `setDisplayLanguage` | `pt`, `en`, `es`, `fr`, `de`, `zh` |
| `setTemperatureUnit` | `celsius`, `fahrenheit` |

A recepção HTTP pelo Orion não comprova execução pela placa. A confirmação depende do resultado e do estado devolvido pelo firmware.

## 9. Executar no Wokwi

1. Abra https://wokwi.com/projects/479153212294129665.
2. Confira se o projeto possui `sketch.ino`, `diagram.json` e `libraries.txt`.
3. Compare com os arquivos da pasta local `firmware/wokwi/`.
4. Atualize os três arquivos se o projeto remoto estiver desatualizado.
5. Confira o broker em `STARLIGHT_MQTT_BROKER`.
6. Confira `DEVICE_ID = "wokwi001"`.
7. Use a rede virtual `Wokwi-GUEST`, sem senha, nas constantes do perfil Wokwi.
8. Provisione `wokwi001` no dashboard.
9. Inicie a simulação e acompanhe o Monitor Serial.

O sketch local examinado contém as constantes de Wi-Fi do perfil Wokwi vazias. Preencha-as antes de executar e não presuma que o gerador configurou a rede virtual automaticamente.

### Diferenças de conexão

| Sinal | Wokwi | Físico |
|---|---:|---:|
| Buzzer | GPIO 18 | GPIO 13 |
| RFID SCK | GPIO 14 | GPIO 18 |
| RFID MOSI | GPIO 13 | GPIO 23 |
| RFID RST | GPIO 17 | GPIO 33 |
| DHT | DHT22, GPIO 4 | DHT11, GPIO 4 |

RGB, LDR, LCD, SS e MISO mantêm os GPIOs indicados na tabela física.

### Interagir com os sensores

- Clique no DHT22 e altere temperatura e umidade.
- Clique no módulo LDR e varie a iluminação.
- Aguarde o próximo ciclo de leitura e publicação.
- Compare a mudança no Serial, nos cards e posteriormente no histórico.
- Provoque uma grandeza por vez para distinguir os sons.
- Retorne à faixa normal e confira a resolução.

No circuito Wokwi local, o RC522 possui UID de simulação `DE:AD:BE:EF`, correspondente a `DEADBEEF` na lista do firmware.

A simulação não comprova níveis elétricos, corrente, resistores, ruído analógico ou qualidade do encapsulamento. O link remoto não pôde ser inspecionado nesta revisão; o procedimento se baseia nos arquivos locais.

## 10. Operar o dashboard

### Monitoramento

Selecione o dispositivo e acompanhe temperatura, umidade, luminosidade e estado. O firmware publica aproximadamente a cada três segundos; o dashboard consulta o estado a cada dois segundos.

Um valor antigo não deve ser interpretado como nova medição. O sistema utiliza heartbeat para verificar atividade; o código define janela de 15 segundos para essa avaliação. Em falha do DHT, pode haver valores anteriores ainda disponíveis, por isso confira também `sensorStatus`.

### Histórico

Escolha um período predefinido ou intervalo personalizado. As consultas são feitas ao STH-Comet, com atualização periódica de cinco segundos.

Os períodos de uma hora, seis horas, 24 horas e sete dias consultam, respectivamente, até 60, 100, 100 e 100 amostras por grandeza no código revisado. Intervalo personalizado e todo o período solicitam até 1000. São limites de consulta, não garantia de cobertura completa de cada janela.

A opção de todo o período não implementa uma exportação integral paginada de todo o banco.

### Triggers

No modo especialista, ajuste mínimo e máximo de cada parâmetro. Mantenha mínimo menor ou igual ao máximo.

| Grandeza | Faixa padrão |
|---|---|
| Temperatura | 10 a 16 °C |
| Umidade | 60 a 80% |
| Luminosidade relativa | 0 a 70% |

Os valores são armazenados por dispositivo no `localStorage` do navegador. Temperatura é mantida internamente em Celsius, mesmo quando exibida em Fahrenheit.

**Limitação operacional:** editar esses campos altera a classificação e a apresentação no dashboard. O código examinado não envia esses limites à ESP32. Para alterar os limites locais do alarme, edite as constantes do firmware-base, gere as variantes e grave novamente a placa.

### Preferências e relatório

O idioma do dashboard é independente do idioma enviado ao LCD. A unidade do visor também é configurada por dispositivo. O LCD HD44780 não exibe caracteres chineses completos; o perfil `zh` utiliza rótulos ASCII no dispositivo.

Use a exportação CSV para obter as amostras carregadas e ocorrências do período. Parte das ocorrências é persistida no navegador; limpar seus dados pode removê-las. Isso não equivale a apagar o histórico MongoDB, nem constitui registro central de auditoria de todas as ações.

## 11. Alertas e reconhecimento RFID

### Avaliação local

O firmware compara as medições às constantes:

```cpp
TEMP_MIN_C = 10.0
TEMP_MAX_C = 16.0
HUMIDITY_MIN = 60.0
HUMIDITY_MAX = 80.0
LUMINOSITY_MIN = 0
LUMINOSITY_MAX = 70
```

Valor igual à fronteira permanece dentro da faixa. Valores abaixo do mínimo ou acima do máximo ativam o bit correspondente.

| Condição | RGB | Frequência | Padrão em ciclo de 1400 ms |
|---|---|---:|---|
| Temperatura | Vermelho | 900 Hz | 0 a 220 ms e 360 a 580 ms |
| Umidade | Azul | 1350 Hz | 0 a 650 ms |
| Luminosidade | Amarelo | 1900 Hz | 0 a 120, 260 a 380 e 520 a 640 ms |
| Manual remoto | Magenta | 700 Hz | 0 a 450 ms |
| Normal | Verde | Sem alarme | Contínuo enquanto normal |

A saída alterna entre condições ativas a cada 1800 ms. O RGB mantém a cor da condição selecionada; o padrão intermitente acima refere-se ao buzzer.

### Controle remoto

No modo especialista, o controle de alerta envia `setAlertMode` pelo Orion. O IoT Agent encaminha o comando MQTT à ESP32, que publica seu resultado.

`on` ativa o alarme manual. `off` desativa o componente manual; se houver alerta ambiental local, o firmware reconhece a condição e a silencia temporariamente.

Não confunda esse botão com uma automação disparada pelos triggers. A automação exigida pelo CP5, com LED azul piscante até a normalização, não foi identificada nesta revisão.

### Cadastrar cartões

1. Ligue a placa e abra o Serial.
2. Aproxime o cartão para obter o UID, inclusive pelo log de acesso negado.
3. Normalize o UID em hexadecimal, sem espaços ou separadores, em maiúsculas.
4. No código-base, encontre `AUTHORIZED_TAGS`.
5. Adicione UID e nome:

```cpp
{"UID_HEXADECIMAL_REAL", "Nome do operador"},
```

6. Gere as variantes.
7. Grave novamente a placa ou atualize o sketch Wokwi.
8. Teste a credencial.

O cadastro de tags nesta versão é feito no firmware, não por uma tela de administração de credenciais.

### Usar cartões

- **Tag autorizada com alarme:** associa operador e UID, registra reconhecimento e pausa RGB e buzzer por 60 segundos.
- **Tag autorizada sem alarme:** registra uma inspeção.
- **Tag desconhecida:** registra tentativa e emite feedback de rejeição; não autoriza a pausa.
- **Condição persistente após a pausa:** o alarme retorna.
- **Nova anomalia durante a pausa:** pode reativar a saída se não fazia parte do conjunto reconhecido.
- **Condição normalizada:** o firmware publica resolução, atribuindo ao Sistema quando não houve reconhecimento associado.

O feedback de autorização usa 2200 Hz por 120 ms; o de rejeição usa 350 Hz por 500 ms no código. O tratamento contínuo dos alarmes também atua sobre o buzzer, portanto a reprodução física deve ser observada durante o teste.

Reconhecer o alarme não remove a causa ambiental. A pausa de toda a indicação visual também precisa ser revista para atender ao requisito acadêmico de LED azul persistente.

## 12. Consultas HTTP para diagnóstico

Os exemplos abaixo são de leitura. O uso normal permanece pelo dashboard, sem depender de Postman.

### Estado atual no Orion

Na EC2:

```bash
curl -i 'http://localhost:1026/v2/entities/urn:ngsi-ld:WineCellar:garrafa011?type=WineCellar' \
  -H 'fiware-service: smart' \
  -H 'fiware-servicepath: /'
```

Espere uma entidade com `id`, `type` e atributos quando cadastro e telemetria estiverem funcionais. Um 404 pode indicar entidade ausente, ID errado, tipo errado ou namespace diferente.

### Histórico no STH-Comet

```bash
curl -i 'http://localhost:8666/STH/v1/contextEntities/type/WineCellar/id/urn%3Angsi-ld%3AWineCellar%3Agarrafa011/attributes/temperature?lastN=10' \
  -H 'fiware-service: smart' \
  -H 'fiware-servicepath: /'
```

O retorno histórico utiliza a estrutura de respostas de contexto e valores com seus horários. O dashboard extrai as amostras para os gráficos.

Para outras grandezas, troque `temperature` por `humidity` ou `luminosity`. Para a simulação, utilize a entidade de `wokwi001`.

O filtro personalizado usa `dateFrom` e `dateTo` com timestamps ISO 8601, além de `lastN`. Consulta vazia exige verificar a assinatura, o namespace e a existência de publicações após sua criação.

### Cadastro e assinatura

```bash
curl -i 'http://localhost:4041/iot/devices?limit=100' \
  -H 'fiware-service: smart' \
  -H 'fiware-servicepath: /'

curl -i 'http://localhost:1026/v2/subscriptions' \
  -H 'fiware-service: smart' \
  -H 'fiware-servicepath: /'
```

Confira Device ID, entidade, tipo e URL interna de notificação. Ter dados atuais no Orion não comprova que a assinatura histórica esteja funcionando.

## 13. Solução de problemas

### getaddrinfo ENOTFOUND orion

A mensagem indica falha ao resolver o nome `orion`. Esse nome pode existir apenas como alias na rede Docker. Uma aplicação Node no computador, um worker hospedado ou outro contêiner em rede diferente pode não conhecer esse nome.

1. Identifique qual operação retornou o erro.
2. Se for o proxy acessando Orion externamente, revise `FIWARE_HOST` e `FIWARE_ORION_URL`.
3. Se for o IoT Agent falando com Orion, revise o destino interno do broker de contexto e a rede Docker.
4. Se a falha aparecer no cadastro, confira o destino registrado no grupo de serviço.
5. Verifique os nomes ou aliases reais no Compose.
6. Atualize a configuração correta e teste novamente.

Salvar o IPv4 do dashboard não corrige necessariamente um destino interno antigo já registrado no IoT Agent.

### IPv4 salvo, mas um ou mais serviços não responderam

A gravação do endereço ocorreu, mas pelo menos um teste HTTP falhou. Isso não comprova que o IP esteja errado nem que todos os serviços estejam indisponíveis.

Teste separadamente Orion, IoT Agent e STH-Comet. Na EC2, examine contêineres e logs. Do computador, verifique acesso às portas e Security Group. Depois teste MQTT de forma separada.

Confira também overrides de URL, host sem `http://` no campo de IPv4 e eventual configuração antiga da hospedagem.

### ETIMEDOUT

A conexão não terminou dentro do prazo. Verifique instância ligada, IP atual, portas publicadas, regras de rede e rotas. Timeout ocorre antes de uma resposta HTTP válida e não deve ser tratado automaticamente como CORS.

### ECONNREFUSED

O destino recusou a conexão. Verifique serviço escutando na porta, mapeamento Docker e processo parado. Compare acesso local na EC2 e externo.

### MQTT conecta, mas o dashboard fica sem dados

Confira `DEVICE_ID`, API key `TEF`, tópicos, provisionamento, atributos do IoT Agent, service e service path. Veja se a entidade recebe atualizações no Orion.

Duas placas com o mesmo cliente MQTT ou Device ID podem disputar conexão e produzir resultados confusos. Use identidades diferentes.

### Wi-Fi ou MQTT não conecta

Confira rede de 2,4 GHz, credenciais, broker configurado, alcance TCP 1883 e ausência de portal de autenticação. No Wokwi, configure explicitamente a rede virtual e confirme acesso ao broker pela internet.

### Dados atuais existem, mas não há histórico

Confira STH-Comet, MongoDB histórico, assinatura Orion, destino `/notify`, formato `legacy`, metadados e headers. Gere novas leituras após corrigir o fluxo. A assinatura não recupera automaticamente dados antigos que nunca foram persistidos.

### DHT retorna NaN ou dht_error

Confira tipo de sensor, DATA, alimentação, GND e pull-up. O intervalo configurado é 2500 ms. Não interprete um último valor válido mantido na entidade como prova de recuperação do sensor.

### LCD acende sem texto

Ajuste contraste, confira 0x27 e conexões I2C. Verifique biblioteca e níveis elétricos do backpack. Não troque SDA/SCL pelos pinos SPI do RFID.

### RFID não reconhece

Confira alimentação 3,3 V, SS, RST, SCK, MISO e MOSI do perfil correto. Verifique UID sem separadores e regrave após alterar a lista.

### Alterar trigger não muda o buzzer

Esse comportamento é compatível com a implementação examinada. O dashboard e o firmware utilizam conjuntos de limites independentes. Modificar o trigger visual não atualiza as constantes da placa.

## 14. Roteiro de validação e evidências

Use uma condição normal inicial, como 13 °C, 70% de umidade e luminosidade relativa abaixo de 70%. A luminosidade precisa ser conferida no valor publicado, não apenas na posição do controle simulado.

| Teste | Procedimento | Resultado a observar |
|---|---|---|
| Infraestrutura | Testar as três APIs e MQTT | Serviços alcançáveis |
| Cadastro | Provisionar um perfil | Registro IoT Agent e entidade coerentes |
| Telemetria | Alterar cada sensor | Valor novo no Orion e dashboard |
| Histórico | Aguardar publicações e consultar STH | Amostras com horários |
| Temperatura | Usar valor maior que 16 °C | Vermelho e dois bipes no perfil local |
| Umidade | Usar valor menor que 60% | Azul e bipe longo |
| Luz | Obter percentual maior que 70% | Amarelo e três bipes |
| Simultaneidade | Manter duas condições anormais | Alternância entre padrões |
| Normalização | Retornar todas as variáveis à faixa | Verde e fim do alarme local |
| Controle remoto | Enviar on pelo dashboard | Execução confirmada e padrão manual |
| RFID | Aproximar tag autorizada em alarme | Operador/UID e pausa |
| Retomada | Manter causa por mais de 60 segundos | Alarme volta |
| Tag desconhecida | Aproximar UID não cadastrado | Registro de acesso negado |
| Preferências | Alterar LCD por dispositivo | Idioma/unidade sincronizados |
| Trigger visual | Alterar limite no navegador | Classificação visual muda |
| Reconexão | Desconectar e reconectar a placa | Estado e novas publicações |

O teste específico exigido de LED azul piscante comandado automaticamente pelo front-end ainda não pode ser declarado aprovado pelo código revisado. Ele deve ser implementado e demonstrado antes de afirmar conformidade integral.

### Evidências da entrega

- Fotografias da montagem e encapsulamento.
- Captura do cadastro do dispositivo.
- Captura com as três grandezas e estado online.
- Resposta STH e gráficos correspondentes.
- Demonstração dos triggers e de sua relação real com os alarmes.
- Sons diferenciados e comportamento simultâneo.
- Registro do operador RFID e pausa.
- Link do vídeo demonstrativo, ainda não fornecido.
- Pitch e hands-on com DHT11, LDR, buzzer e ESP32 físicos.

O enunciado atribui 10% ao encapsulamento, 10% ao diferencial, 10% ao manual, 20% ao front-end, 10% ao GitHub e 40% ao hands-on. A data informada para a entrega é 30/10/2026.

## 15. Manutenção e encerramento

Para alterações permanentes no firmware, edite o código-base, gere os perfis e regrave os dispositivos. Preserve os dados de configuração e credenciais fora do repositório público.

Para verificar o software:

```bash
npm run lint
npm test
npm run build:site
npm run validate:site
```

Os testes existentes não foram executados novamente durante a produção destes documentos. A compilação Arduino, a simulação online e o hands-on também precisam de validação no ambiente correspondente.

Para interromper o Vite, use Ctrl+C no terminal. Para parar a infraestrutura sem remover volumes:

```bash
docker compose stop
```

Para iniciar novamente:

```bash
docker compose start
```

Antes de encerrar o laboratório AWS, salve as alterações necessárias e registre as evidências. Na próxima sessão, confira novamente o IPv4 público.

## 16. Segurança e limites de uso

Mantenha MongoDB interno, restrinja portas e proteja chaves SSH. A configuração MQTT sem credenciais e os endpoints HTTP de prova de conceito não oferecem, sozinhos, as proteções necessárias a uma implantação de produção.

UID RFID identifica uma credencial cadastrada no protótipo e não comprova identidade pessoal com resistência à clonagem. Os eventos no navegador não são uma trilha de auditoria imutável.

As frequências e GPIOs deste manual foram obtidos do código local. Recomendações de proteção elétrica são cuidados de montagem, não evidência de que a bancada já os utiliza. Não foram presumidos RTC, EEPROM externa, aplicativo móvel ou backend Python.

## 17. Referências e autoria

- [FIWARE Descomplicado, Prof. Fábio Henrique Cabrini](https://github.com/fabiocabrini/fiware).
- [Repositório StarLight](https://github.com/EstelaMariano/Vinheria-Full).
- [Wokwi informado pela equipe](https://wokwi.com/projects/479153212294129665).
- Enunciado CP5 e imagem de arquitetura fornecidos pela equipe.
- Arquivos locais de firmware, dashboard, configuração e circuito examinados nesta revisão.

**Equipe:** Beatriz Soares Salve, RM 568791; Estela Mariano da Silva, RM 569513; Gabriela Correa Pinon Labrada, RM 569849; Lucca Savoia Bergamasco Diniz, RM 569489.

