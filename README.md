# Limpeza de dados de Poços Perfurados da ANP

A ANP disponibiliza, em arquivos csv, os dados de produção por poço desde o ano de 2010, sendo que, neste ano, os dados foram divulgados com o histórico do quinquênio anterior (2005-2009). Os dados estão disponíveis [neste link](https://www.gov.br/anp/pt-br/centrais-de-conteudo/dados-abertos/producao-de-petroleo-e-gas-natural-por-poco).

Este projeto visa unificar os procedimentos necessários para a criação de uma base limpa para inserção em um DataLake (público ou privado) e fornecer subsídios para análises do setor, bem como visualizações com atualizações periódicas

# SQLModel

Para gerar os dados com os relacionamentos necessários, incluindo com os dicionários (sejam os dos diretórios da Base dos Dados ou os locais), optei por usar o projeto [SQLModel](https://github.com/tiangolo/sqlmodel) ([documentação](https://sqlmodel.tiangolo.com/)), do mesmo autor do [FastAPI](https://fastapi.tiangolo.com/).

## Instalação
```bash
pip install sqlmodel
```