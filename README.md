# SD-arquitetura-importacao-de-devedores

## O problema

Você está entrando em uma empresa de cobrança que processa milhões de interações por mês. Um dos sistemas mais críticos é o de importação de devedores, que alimenta toda a operação.

A empresa recebe dados de clientes (credores) via SFTP em dois fluxos:

**Carga semanal (base completa)**
- Disponibilizado todo domingo às 02h
- Base completa de devedores: ~10 milhões de registros
- Formato CSV
- Precisa estar pronto antes das 08h de segunda

**Atualizações horárias (baixas de pagamento)**
- Arquivos de hora em hora, das 08h às 20h
- Apenas atualizações: pagamentos, acordos, baixas
- ~5.000 registros por arquivo
- Precisa processar em até 15 minutos

## Algumas coisas que você precisa saber

- O SFTP do cliente é instável. Cai, dá timeout, a conexão trava.
- O cliente já mandou o mesmo arquivo duas vezes por engano. Mais de uma vez.
- Segunda-feira às 08h, se a base não estiver atualizada, o time de operações não consegue trabalhar.
