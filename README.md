<h1 align="center">📚 Edihub Totem Escolar</h1>
<p align="center"><b>Painel IoT interativo de editais nas escolas</b></p>

<p align="center">
  <img src="https://img.shields.io/badge/ESP32-Wokwi-E7352C?logo=espressif&logoColor=white" alt="ESP32">
  <img src="https://img.shields.io/badge/IoT-ThingSpeak-0076A8" alt="ThingSpeak">
  <img src="https://img.shields.io/badge/Edge-Computing-2E7D32" alt="Edge Computing">
  <img src="https://img.shields.io/badge/ODS-4%20%7C%2010-C5192D" alt="ODS 4 e 10">
</p>

<p align="center">
  <b>Global Solution 2026.2 · CitizenTech · Edge Computing & Computer Systems</b><br>
  FIAP · Engenharia de Software · 1º ano · Rio de Janeiro
</p>

Dispositivo IoT baseado em **Edge Computing** que alerta estudantes sobre prazos de vestibulares e concursos militares diretamente nos corredores das escolas, além de medir o engajamento presencial dos alunos.

<p align="center">
  <a href="https://wokwi.com/projects/480972212943588353"><b>🔌 Abrir simulação no Wokwi</b></a> ·
  <a href="https://thingspeak.mathworks.com/channels/3584126"><b>📊 Canal público no ThingSpeak</b></a> ·
</p>

---

## 👥 Integrantes

| Nome completo | RM |
| :--- | :--- |
| Henrique Nunes Mororó | 574073 |
| Bernardo de Paula Rodrigues | 572376 |
| Arthur Moreira Costa | 571532 |

---

## 1. Descrição do problema

Todo ano, milhares de estudantes perdem **prazos de isenção de taxa e de inscrição** em vestibulares (UERJ, UFRJ, ENEM) e concursos militares (EsPCEx, IME). Essas datas são divulgadas em editais longos, em PDF, espalhados pelos sites de cada instituição.

O problema é maior para alunos de **escolas públicas e comunidades** que não têm celular próprio ou internet constante em casa. Para eles, a informação simplesmente não chega a tempo. Perder uma isenção pode significar não fazer a prova por não conseguir pagar a taxa.

## 2. Objetivo da solução

Levar os prazos mais urgentes **para dentro da escola**, em um totem físico instalado no corredor, que:

- mostra o certame com o prazo mais próximo de terminar;
- alerta de forma visual e sonora quando faltam menos de 24 horas;
- continua funcionando mesmo sem internet;
- envia para a nuvem dados de engajamento (quantos alunos se aproximam e quais certames são consultados), ajudando a escola e o Edihub a entender o interesse dos estudantes.

## 3. Cidadão / público beneficiado

- **Estudantes do ensino médio e de cursinhos**, principalmente de escolas públicas e comunitárias, sem acesso contínuo a celular ou internet.
- **Escolas e cursinhos conveniados**, que passam a ter dados sobre o interesse dos alunos em cada processo seletivo.
- **Famílias**, que deixam de depender de o aluno encontrar sozinho a informação em editais extensos.

## 4. ODS relacionados

| ODS | Como o projeto contribui |
| :--- | :--- |
| **ODS 4 — Educação de Qualidade** | Amplia o acesso ao ensino superior e técnico, evitando que estudantes percam inscrições e isenções. |
| **ODS 10 — Redução das Desigualdades** | Leva a mesma informação a quem não tem celular ou internet, diminuindo a desvantagem de alunos de baixa renda. |

---

## 5. Funcionamento do sistema

### Arquitetura

```
┌──────────────────┐   HTTPS GET    ┌──────────────────────────┐   HTTP GET    ┌──────────────────┐
│  API simulada    │  (10 em 10 min)│      ESP32 (BORDA)       │ (20 em 20 s)  │    ThingSpeak    │
│  db.json GitHub  │ ─────────────▶ │ • lê sensores            │ ────────────▶ │ Field 1: aproxim.│
│  (certames)      │                │ • calcula prazos (NTP)   │               │ Field 2: certame │
└──────────────────┘                │ • escolhe o mais urgente │               │ Field 3: luz (%) │
                                    │ • decide LED/alarme/tela │               └──────────────────┘
   Sem internet?  ───────────────▶  │ • usa lista OFFLINE      │
                                    └──────────────────────────┘
     Entradas: HC-SR04 · LDR · Botão          Saídas: OLED · LED RGB · Buzzer
```

### Passo a passo do funcionamento

1. **Inicialização:** o ESP32 conecta ao Wi-Fi, sincroniza o relógio via NTP (horário de Brasília) e baixa a lista de certames da API simulada (`db.json`). Se a API não responder, carrega a **lista de contingência offline** gravada no código.
2. **Processamento local (Edge Computing):** para cada certame da categoria selecionada, o ESP32 calcula quantas horas faltam até o fim do prazo, descarta os vencidos e escolhe o **mais urgente**. Nenhuma dessas decisões depende da nuvem.
3. **Exibição:** o display OLED mostra a categoria, o nome do certame, o evento (isenção, inscrição, resultado) e quantos dias faltam. O rodapé indica se os dados vieram da API (`[API]`) ou da reserva offline (`[OFF]`).
4. **Presença:** quando o sensor ultrassônico detecta um aluno a menos de 1 metro, o totem conta uma aproximação, registra o certame consultado e, se o prazo estiver nas últimas 24 h, emite um bipe.
5. **Interação:** o botão alterna entre **VESTIBULAR** e **MILITAR**.
6. **Economia de energia:** o LDR mede a luz do corredor. Abaixo de 30% de luminosidade, o contraste da tela é reduzido.
7. **Telemetria:** a cada 20 segundos, o ESP32 envia ao ThingSpeak o total de aproximações do dia, o último certame consultado e o nível de luz. A contagem diária zera na virada do dia.

### Condições estabelecidas (regras de resposta)

| Condição | Resposta do dispositivo |
| :--- | :--- |
| Distância < 100 cm | Rodapé “Ola, estudante!”, +1 aproximação, registra certame consultado |
| Prazo > 24 h | LED **verde** + “Termina em X dias” |
| Prazo ≤ 24 h | LED **vermelho piscando** + “ULTIMAS 24H!” + bipe ao detectar aluno |
| Sem dados ou relógio não sincronizado | LED **azul** |
| Luz ambiente < 30% | Contraste do OLED reduzido (10 de 255) |
| Botão pressionado | Alterna VESTIBULAR ↔ MILITAR |
| API indisponível | Usa lista offline e mostra `[OFF]` |

---

## 6. Componentes utilizados

| Componente | Função | Pino ESP32 | Observação |
| :--- | :--- | :--- | :--- |
| **ESP32 DevKit C V4** | Microcontrolador, Wi-Fi e processamento local | — | — |
| **Display OLED SSD1306** | Exibe certame, prazo e status | `GPIO 21` (SDA) / `GPIO 22` (SCL) | I2C, endereço `0x3C` |
| **Sensor ultrassônico HC-SR04** *(sensor 1)* | Detecta presença do aluno (< 100 cm) | `GPIO 5` (TRIG) / `GPIO 18` (ECHO) | Alimentação 5V |
| **Sensor de luz LDR** *(sensor 2)* | Mede a luz ambiente (ajuste de brilho) | `GPIO 34` (AO) | Canal ADC1, compatível com Wi-Fi |
| **LED RGB (cátodo comum)** | Sinalização de urgência | `GPIO 25` (R) / `26` (G) / `27` (B) | Resistor de 220 Ω por canal |
| **Buzzer** | Alerta sonoro para prazos < 24 h | `GPIO 13` | Toca só com presença detectada |
| **Pushbutton** | Alterna VESTIBULAR / MILITAR | `GPIO 4` | `INPUT_PULLUP` |

**Bibliotecas:** `WiFi.h`, `WiFiClientSecure.h`, `HTTPClient.h`, `ArduinoJson`, `Adafruit SSD1306`, `Adafruit GFX Library`.

**Nuvem:** ThingSpeak (requisições HTTP REST para `api.thingspeak.com/update`).

| Campo ThingSpeak | Dado | Finalidade |
| :--- | :--- | :--- |
| Field 1 — Aproximacoes | Aproximações de alunos no dia | Medir o tráfego no totem |
| Field 2 — Certame consultado | ID do último certame visto | Identificar os certames de maior interesse |
| Field 3 — Luz ambiente | Luminosidade (0 a 100%) | Acompanhar a economia de energia |

---

## 7. Estrutura do repositório

```
Edge-gs-Edihub/
├── README.md          → esta documentação
├── main.ino           → código-fonte completo do ESP32
├── diagram.json       → circuito do Wokwi
├── libraries.txt      → bibliotecas usadas no Wokwi
├── db.json            → cópia da API simulada (certames)
└── LICENSE            → licença MIT
```

---

## 8. Como executar

### Opção rápida
Abra o link do **[projeto no Wokwi](https://wokwi.com/projects/480972212943588353)** e clique em **▶ Start Simulation**. O projeto já está configurado com a API e o canal do ThingSpeak.

### Recriar do zero
1. **API simulada:** hospede o `db.json` em um repositório **público** no GitHub e copie a URL **Raw**.
2. **ThingSpeak:** crie um canal com 3 campos (Aproximacoes, Certame consultado, Luz ambiente) e copie a **Write API Key**.
3. **Wokwi:** crie um projeto **ESP32 (Arduino)** e:
   - cole o `diagram.json` na aba de mesmo nome;
   - crie o arquivo `libraries.txt` com `Adafruit SSD1306`, `Adafruit GFX Library` e `ArduinoJson`;
   - cole o `main.ino` na aba `sketch.ino`;
   - altere `URL_API` e `THINGSPEAK_KEY` no topo do código;
   - inicie a simulação.

> ⚠️ O display OLED não exibe acentos. Por isso os textos no `db.json` são escritos sem acentuação (ex.: `Isencao`). As datas dos certames são **fictícias**, usadas apenas para demonstração.

---

## 9. Demonstração: alterando os sensores simulados

Com a simulação rodando, clique em cada componente do Wokwi para alterar seu valor:

| Teste | Ação no Wokwi | Resposta esperada |
| :--- | :--- | :--- |
| Presença | Arrastar a distância do HC-SR04 para < 100 cm | “Ola, estudante!”, contador de aproximações sobe |
| Nova aproximação | Afastar (> 100 cm) e aproximar novamente | Cada aproximação soma +1 (ficar parado não conta) |
| Prazo crítico | Certame com prazo no dia atual + aproximação | LED vermelho piscando, “ULTIMAS 24H!”, bipe |
| Prazo normal | Certame com vários dias restantes | LED verde e “Termina em X dias” |
| Categoria | Clicar no botão azul | Alterna VESTIBULAR ↔ MILITAR |
| Luz ambiente | Mover o controle do LDR para o escuro | Brilho reduzido, Field 3 cai no ThingSpeak |
| Modo offline | Alterar `URL_API` para um endereço inválido | Mensagem “Usando reserva OFFLINE” e `[OFF]` no display |

---

## 10. Evidências de funcionamento

### 10.1 Circuito montado no Wokwi

<p align="center">
  <a href="https://wokwi.com/projects/480972212943588353">
    <img src="https://github.com/user-attachments/assets/993b442c-d88f-4d94-b3bf-27220d071616" alt="Circuito do Edihub Totem Escolar no Wokwi: ESP32 ligado ao HC-SR04, LDR, OLED, LED RGB, buzzer e botão" width="820">
  </a>
  <br>
  <sub>ESP32 DevKit C V4 ligado ao HC-SR04, LDR, display OLED, LED RGB, buzzer e botão. Clique na imagem para abrir a simulação.</sub>
</p>

### 10.2 Resposta do dispositivo às condições

<table>
  <tr>
    <th width="50%">✅ Prazo normal (LED verde)</th>
    <th width="50%">🚨 Prazo crítico (LED vermelho)</th>
  </tr>
  <tr>
    <td><img src="https://github.com/user-attachments/assets/d4c20743-b32e-4a94-a639-a3c0b671ff81" alt="Simulação com LED verde e Serial Monitor mostrando API carregada, presença detectada e envio ao ThingSpeak" width="100%"></td>
    <td><img src="https://github.com/user-attachments/assets/728d1d6f-bb74-4e54-85b2-a00488d87048" alt="Simulação na categoria militar com LED vermelho e alerta de últimas 24 horas" width="100%"></td>
  </tr>
  <tr>
    <td><sub>API carregada (<code>[API] 5 certames carregados</code>), aluno detectado a 2 cm (<code>[PRESENCA]</code>) e telemetria aceita pelo ThingSpeak (<code>HTTP 200</code>).</sub></td>
    <td><sub>Após o botão alternar para <b>MILITAR</b>, o certame mais urgente fica a menos de 24 h: LED vermelho piscando, aviso “ULTIMAS 24H!” e bipe.</sub></td>
  </tr>
</table>

### 10.3 Telemetria no ThingSpeak

<p align="center">
  <a href="https://thingspeak.mathworks.com/channels/3584126">
    <img src="https://github.com/user-attachments/assets/cccc592f-614a-443d-a364-e0655fbc3db3" alt="Gráficos do canal Global Solution Edge Computing no ThingSpeak com os campos Aproximacoes, Certame consultado e Luz ambiente" width="820">
  </a>
  <br>
  <sub>Canal público <b>Global Solution Edge Computing</b> (ID 3584126) recebendo os 3 campos enviados pelo ESP32. Clique na imagem para ver os dados ao vivo.</sub>
</p>

---

## 11. Por que é Edge Computing?

Todo o processamento que gera a resposta ao aluno acontece **no próprio ESP32, na borda da rede**:

- cálculo dos dias e horas restantes;
- escolha do certame mais urgente;
- acionamento do LED, do buzzer e do brilho da tela.

A nuvem recebe apenas a **telemetria resumida**, o que reduz o tráfego e a latência. O totem continua útil **mesmo sem conexão**, graças à lista offline, e isso é essencial em escolas com internet instável.
