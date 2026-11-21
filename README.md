# Caixa de Remédios 2.0 — Aplicativo Android

Aplicativo Android desenvolvido em Java como parte de um **Projeto Integrador** de nível médio no **IFES** (Instituto Federal do Espírito Santo). Ele faz o gerenciamento de medicamentos, agendamento de alarmes e controle remoto de uma caixa de remédios automatizada baseada em ESP32.

O firmware embarcado da caixa está disponível em repositório separado: [erick-schultz/Caixa-de-Remedios-2.0](https://github.com/erick-schultz/Caixa-de-Remedios-2.0).

## Funcionalidades

- **Gestão de 3 compartimentos:** cadastro e edição de nome do medicamento, quantidade disponível, descrição, data de vencimento, horário inicial e frequência de uso.
- **Agendamento de alarmes recorrentes:** intervalos de `1h`, `3h`, `6h`, `12h`, `18h`, `24h`, `36h`, `48h` ou `72h` (além da opção `TESTE`, com intervalo de 10 segundos, usada para validação em bancada).
- **Acionamento sincronizado com a caixa:** quando o alarme dispara, o celular vibra, abre a tela de aviso com o nome do remédio e envia comandos HTTP ao ESP32 para acender os LEDs e tocar o buzzer.
- **Abertura e fechamento pela rede local:** controle direto dos servomotores de cada compartimento pela tela principal. Ao confirmar o fechamento da caixa, a quantidade restante do medicamento é decrementada automaticamente.
- **Configuração de IP no próprio app:** o endereço IP do ESP32 na rede local é definido pelo menu de configurações na barra superior e salvo localmente.

## Arquitetura e tecnologias

- **Linguagem:** Java (Java 8)
- **Plataforma:** Android (`minSdk 24` / Android 7.0, `compileSdk` e `targetSdk 34`)
- **Build:** Gradle `8.2` / Android Gradle Plugin `8.2.2`
- **Componentes utilizados:**
  - `AndroidViewModel` e `LiveData` (`MainActivityViewModel`, `NewItemActivityAlarmViewModel`) para desacoplar as chamadas de rede da UI
  - `ExecutorService` para executar requisições HTTP fora da thread principal
  - `HttpURLConnection` (`HttpRequest` e `Repository`) com timeout de `3000 ms`
  - `AlarmManager` + `BroadcastReceiver` (`AlarmReceiver`) para disparo dos horários agendados
  - `SharedPreferences` (`Config`) armazenando os compartimentos em JSON (`caixa1`, `caixa2`, `caixa3`) e o IP do ESP32 (`esp32address`)

## Integração com o ESP32

O aplicativo se comunica diretamente com o servidor HTTP rodando na porta `80` do ESP32 ([Caixa-de-Remedios-2.0](https://github.com/erick-schultz/Caixa-de-Remedios-2.0)), na mesma rede Wi-Fi local.

| Ação no aplicativo | Requisição enviada | Comportamento na caixa |
| :--- | :--- | :--- |
| Botão **Abrir Caixa 1/2/3** | `GET /bot1`, `/bot2`, `/bot3` | Move o servo do compartimento para a posição aberta |
| Botão **Fechar Caixa 1/2/3** | `GET /close1`, `/close2`, `/close3` | Fecha o compartimento e apaga os LEDs; o app reduz a quantidade em 1 |
| Disparo do alarme | `GET /ledH` | Aciona os relés de iluminação |
| Disparo do alarme (após 300 ms) | `GET /buzzerH` | Toca o alarme sonoro no buzzer |

Todas as requisições esperam `HTTP 200 OK` com retorno `1` seguido de quebra de linha (`"1\n\n"`) para confirmar que o comando foi executado pela placa. O `AndroidManifest.xml` mantém `android:usesCleartextTraffic="true"` justamente para permitir o tráfego HTTP local com o microcontrolador.

## Estrutura do projeto

```text
app/src/main/java/queiroz/larissa/projetointegrador/
├── activity/
│   ├── MainActivity.java            # Tela principal dos 3 compartimentos, menu de IP e agendamento
│   ├── NewItemActivity.java         # Cadastro e edição de medicamento/alarme por compartimento
│   ├── AlarmReceiver.java           # BroadcastReceiver que vibra e abre a tela de alarme
│   └── NewItemActivityAlarm.java    # Tela de alerta que aciona LEDs e buzzer no ESP32
├── model/
│   ├── Compartimento.java           # Modelo dos dados do medicamento e cálculos de horário/estoque
│   ├── MainActivityViewModel.java   # Chamadas assíncronas de abrir/fechar compartimentos
│   ├── NewItemActivityAlarmViewModel.java # Chamadas assíncronas de LED e buzzer
│   └── Repository.java              # Montagem das rotas HTTP para o ESP32
└── util/
    ├── Config.java                  # Leitura e gravação em SharedPreferences (JSON + IP)
    ├── HttpRequest.java             # Cliente HTTP baseado em HttpURLConnection
    └── Util.java                    # Conversão de streams e utilitários gerais
```

## Como executar

### Pré-requisitos

- **Android Studio** (compatível com AGP `8.2.2` / Gradle `8.2`)
- **JDK 17** (para execução do Gradle no Android Studio)
- Dispositivo físico ou emulador com **Android 7.0 (API 24)** ou superior

### Passos

1. Clone o repositório e abra a pasta raiz no Android Studio:
   ```bash
   git clone https://github.com/larissa-bianca/projetoIntegrador.git
   ```
2. Aguarde a sincronização do Gradle e compile/instale o app no celular.
3. Conecte o celular na mesma rede Wi-Fi do ESP32.
4. No app, toque no ícone de engrenagem no canto superior direito e informe o IP exibido no Monitor Serial do ESP32 (apenas o endereço, ex.: `192.168.0.15`, sem `http://`).
5. Cadastre os medicamentos em cada compartimento pelo botão **Editar**.

## Observações

- **Precisão dos alarmes:** o agendamento usa `AlarmManager.setRepeating(RTC_WAKEUP, ...)`, que pode apresentar pequenos atrasos em versões mais recentes do Android devido às otimizações de economia de bateria do sistema operacional.
- **Permissões:** o app solicita `INTERNET`, `VIBRATE` e `SET_ALARM`. Em versões recentes do Android, também pode ser necessário permitir a exibição sobre outros apps ou desativar otimizações de bateria para que a tela de alarme abra em segundo plano sem bloqueio.
- **Rede local:** se o IP do ESP32 mudar via DHCP ao reiniciar o roteador, basta atualizar o novo IP no menu de configurações do aplicativo.
