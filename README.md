# Contador de Fluxo

Aplicativo web que conta as pessoas que passam por um local usando visão computacional. Ele roda inteiro no navegador: a câmera do aparelho é analisada localmente, e nenhuma imagem é gravada ou enviada.

## O que ele faz

- **Entradas e saídas:** detecta pessoas no vídeo e conta cada uma que cruza uma linha virtual, separando por sentido.
- **Lotação:** mostra a ocupação atual contra um limite configurável, com alerta ao atingir o limite e correção manual (+1 / −1).
- **Painel em tempo real:** gráfico de fluxo por minuto dos últimos 30 minutos e registro de cada passagem.
- **Exportação CSV:** baixa todos os eventos registrados.
- **Webhook:** envia cada evento em JSON para um endereço seu, por exemplo um fluxo n8n que grava no BigQuery.
- **Modo demonstração:** ao abrir, a página simula pessoas passando para mostrar o funcionamento sem câmera.

## Como usar

### No computador

1. Baixe o arquivo `index.html`.
2. Abra no Chrome ou no Edge.
3. Clique em **Iniciar câmera** e autorize o acesso.
4. Arraste as duas pontas azuis para posicionar a linha sobre a passagem. A seta verde mostra o sentido que conta como entrada; use **Inverter sentido** se precisar.

### No celular

O navegador do celular só libera a câmera em páginas https. Publique o repositório com o GitHub Pages:

1. No repositório, abra **Settings → Pages**.
2. Em **Source**, escolha **Deploy from a branch**, branch `main`, pasta `/ (root)`.
3. Acesse `https://rcruzjs.github.io/Aplicativo-contagem-pessoas/`.

## Posicionamento da câmera

- Instale a câmera no alto, apontada para baixo sobre a passagem. Isso reduz a sobreposição entre pessoas.
- Deixe a linha de contagem no meio do quadro, com espaço dos dois lados, para a pessoa ser detectada antes e depois de cruzar.
- Evite contraluz forte (porta de vidro com sol atrás, por exemplo).

## Integração com n8n e BigQuery

Informe o endereço do webhook no painel **Ajustes**. Cada passagem gera um `POST` com `Content-Type: application/json`:

```json
{
  "event_id": "uuid",
  "timestamp": "2026-11-09T14:03:22.120Z",
  "local": "Entrada principal",
  "direcao": "entrada",
  "entradas_total": 12,
  "saidas_total": 7,
  "lotacao": 5,
  "limite": 50,
  "track_id": 31,
  "origem": "contador-web"
}
```

O campo `direcao` vale `entrada` ou `saida`. Eventos que falham no envio ficam na fila e são reenviados a cada 15 segundos. O webhook precisa aceitar requisições de outras origens (CORS).

Tabela sugerida no BigQuery:

```sql
CREATE TABLE fluxo.eventos (
  event_id STRING,
  timestamp TIMESTAMP,
  local STRING,
  direcao STRING,
  entradas_total INT64,
  saidas_total INT64,
  lotacao INT64,
  limite INT64,
  track_id INT64,
  origem STRING
) PARTITION BY DATE(timestamp);
```

## Como funciona

1. **Detecção:** o modelo COCO-SSD (`lite_mobilenet_v2`), executado com TensorFlow.js, localiza as pessoas em cada quadro.
2. **Rastreamento:** cada detecção é associada à pessoa mais próxima do quadro anterior, o que mantém um identificador por pessoa enquanto ela está em cena.
3. **Cruzamento:** quando o centro de uma pessoa rastreada muda de lado em relação à linha, o evento é contado como entrada ou saída. Uma faixa morta em torno da linha evita contagem dupla por tremor.

O TensorFlow.js e o modelo são baixados da internet ao iniciar a câmera. Não há etapa de build nem dependências para instalar.

## Limitações

- A contagem e os ajustes ficam na memória da página. Recarregar zera tudo; exporte o CSV antes.
- A contagem pausa se a aba ficar em segundo plano.
- O modelo é leve e pode errar com grupos muito juntos ou pessoas se cruzando sobre a linha.
- É necessária conexão com a internet ao iniciar a câmera, para baixar o modelo.
- O modo com câmera ainda não foi validado em campo; confira a contagem manualmente nos primeiros usos.

## Privacidade

O vídeo é processado no próprio aparelho. O aplicativo não grava imagens, não faz reconhecimento facial e só transmite os eventos de contagem, e apenas se um webhook for configurado. Em locais abertos ao público, avalie a sinalização de monitoramento por câmera conforme a LGPD.
