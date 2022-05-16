# Limpeza de dados de Poços Perfurados da ANP
A ANP disponibiliza, em arquivos csv, os dados de produção por poço desde o ano de 2010, sendo que, neste ano, os dados foram divulgados com o histórico do quinquênio anterior (2005-2009). Os dados estão disponíveis [neste link](https://www.gov.br/anp/pt-br/centrais-de-conteudo/dados-abertos/producao-de-petroleo-e-gas-natural-por-poco).

## Introdução

O objetivo deste ETL é buscar todos os arquivos, limpar e salvar em um único dataset, particionado no formato Hive, seguindo o padrão da [Base dos Dados](https://basedosdados.org). O particionamento será feito por ano, mes, sigla_uf, id_bacia.

Importante observar os seguintes pontos do material disponibilizado:

- Apesar de serem csv, estes arquivos são apenas uma "representação" de arquivos Excel em um formato supostamente aberto
- A agência mantém a padronização brasileira de números, mantendo nas células dados que seriam apenas de formatação, como o separador de milhar (".", segundo o padrão brasileiro), símbolo de percentual, com seus números inteiros (como "12,95%", ao invés de "0.1295" ou "0,1295")
- As colunas possuem diversos níveis (Multiindex), novamente replicando um padrão utilizado em planilhas de Excel, o que torna ainda mais complexo o tratamento destes dados
- A Agência também faz uma consolidação anual, após fechar o mês de dezembro, consolidando todo o ano em um único arquivo zip, contendo 36 arquivos em cada, correspondentes a:
    - Mês
        - Terra
        - Mar
        - Pré-sal
- Cabe ressaltar que a produção do pré-sal já está incluída nos valores de Mar, ou seja, O ETL ainda terá que localizar, em cada poço, se ele está no pré-sal e utilizar uma flag ``presal`` com True ou False (todos os poços em terra serão false), para não duplicar os valores
- Além disso, nos anos fechados, os aquivos contém, nas células superiores, informações textuais, que "confundem" o pandas e, assim, precisam ser eliminadas

## Primeira tentativa

Comecei tentando fazer um processo como o recomendado pela Base dos Dados, que permitira a replicação em qualquer computador. Verifiquei, durante o processo, que existiam inúmeros erros e inconsistências em função do tratamento manual dos dados. Fazer essa limpeza de forma programática mostrou-se, assim, de grande custo e desnecessária, dado que se tratam de dados históricos.

Dessa forma optei por utilizar os dados já tratados como base até o fim do ano de 2021, com dados já limpos e compatibilizados com este ETL que passará a observar, a partir de agora, os formatos definidos por mim para permitir a atualização mensal com menor esforço.

A partir de agora, passarei a trabalhar com arquivos históricos de 2005 a 2021 e, a partir de 2022, usarei um processo de limpeza manual nos arquivos, inserindo-os em uma pasta de entrada e o processo de limpeza e ingestão de dados se preocupará apenas com a inserção dos dados a partir de 2022.

## Sobre a limpeza dos dados históricos

Listo, aqui, os problemas e soluções encontrados para a unificação dessa base histórica, que formarão o conjunto de arquivos originais em que trabalharei a seguir

### 1. Unificação em arquivos zip

Os dados da ANP são fornecidos de forma "particionada" pelo ano e local (terra/mar). Adiante isso será tratado, porém, deve-se ter em mente que, por conta disso, o primeiro processo é baixar o arquivo zip do mês, descompactar e verificar a nomenclatura.

### 2. Nomenclatura

Não existe um padrão estrito de nomes ao longo dos anos. Ora são usados underlines (``_``), ora hifens (``-``). Também existe o caso do ano de 2015 em que foram utilizados caracteres acentuados (por exemplo ``2015_01_produção_mar.csv``).

Temos, também, a não padronização das caixas alta e baixa nos locais ("Terra" ou "terra", "pre-sal", "PreSal", "presal", etc). Como não existe, no arquivo, a indicação do local ("terra", "mar", "pre-sal") tal informação é relevante para uma correta classificação dos campos.

### 3. Segmentação dos arquivos

Quando a ANP passou a separar mar e pré-sal, foi utilizada uma redundância que está, inclusive, documentada na página da agência:

!!! Tip "ANP"
    Mensalmente são publicados três arquivos, com as listagens dos poços conforme a localização: terra, mar, pré-sal (neste último caso se trata dum subconjunto dos dados de produção em mar, contendo apenas os poços no polígono do pré-sal). 


Assim, não podemos simplesmente adicionar os poços de mar e pré-sal, dado que geraria duplicidade nos dados. Inseri, então, uma coluna de localização e uma de pré-sal (``booleana``) que tratei da seguinte forma:

- poços em terra: ``False`` (todos os lidos nos arquivos de terra)
- poços no mar: ``False`` (todos os lidos nos arquivos de mar)
- poços no pré-sal: ``True`` (o arquivo do pré-sal é lido apenas para alterar a flag ``presal`` para ``True`` nos poços que estiverem nele)

### 4. Formato do header em múltiplos níveis

Os arquivos até 2017 (inclusive) tinham um dicionário que, a partir de 2018, foi estendido. Nestes a ANP usou o header em 2 níveis e, nos anos seguintes, em 3. Isso é uma clara utilização do formato Excel, apenas convertido para csv, que dificulta bastante a leitura pela biblioteca Pandas. Optei, então, em apenas seguir a ordem das colunas, eliminando o header da ANP e recriando-os com os nomes unificados (por exemplo, em "Nome do Poço" existe um nível inferior com as colunas "ANP" e "Operador", que alterei para ``nome_poco_anp`` e ``nome_poco_operdor``)

### 5. Unificação de arquivos de meses incompleta

O "padrão normal" da agência é disponibilizar o arquivo de cada mês ao longo do ano e, uma vez fechado o ano, gerar um único arquivo para cada localização. Isso, porém, também não foi unificado em todos os anos. Existem anos em que não temos os arquivos do mês, enquanto outros possuem apenas estes, sem a unificação anual.

Observando a série histórica, vi que a melhor forma de tratar isso foi o seguinte:

- até o ano de 2018 utilizar os arquivos unificados (anual)
- a partir de 2019 utilizar os arquivos mensais, dado que em 2020 (possivelmente por conta da pandemia de Covid-19) a ANP parou de unificá-los e esta será a forma utilizada daqui para a frente

### 6. Sobre a formatação dos dados nos arquivos

Ao observar o conteúdo dos arquivos percebe-se, como já citado anteriormente, que o processo é inteiramente manual, com pessoas que abrem os arquivos originais no Excel (possivelmente no formato nativo ``xls`` ou ``xlsx``) e, para atender à obrigatoriedade de usar um formato aberto, apenas salva em formato ``csv`` utilizando o padrão ``Salvar como...`` do Excel, o que não é aconselhável, visto que este programa não permite parametrizar os dados (codificação, separador etc.). 

Também detectei que as células são salvas com o conteúdo formatado (por exemplo, percentuais estão inseridos como uma string "12,65%"). Estes casos também precisam ser tratados arquivo por arquivo, já que existem diferenças entre eles. A melhor forma é usar os seguintes padrões:

- separação de decimais com ponto (``.``)
- eliminar os separadores de milhar e não incluir nos arquivos
- salvar com a codificação ``utf-8``
- eliminar sinais de formatação, como ``$`` para moedas ou ``%`` para percentuais
- estes, inclusive, devem usar a notação decimal, ou seja, ``0.1265`` ao invés de ``12,65%``

### 7. Linhas e colunas inseridas para análise e "assinatura"

Como já dito, o processo é todo manual, o que, muitas vezes, leva a um operador inserir algumas colunas ou linhas temporárias (prática bastante comum quando se trabalha com planilhas) e esquecer de eliminá-las. Há casos de totalizações em colunas, inserções de linhas e colunas para facilitar a visualização, etc. Listo, abaixo, o que localizei:

2015:
- os arquivos foram salvos com pontuação no nome, precisando de uma limpeza. O termo ``produção`` deve ser convertido para ``producao``, conforme citado anteriormente

2019:
- 2019_02_producao_terra.csv: colunas inseridas indevidamente
    - Os valores nas colunas Etano e Grau API (locais onde foram indevidamente inseridos) foram excluídos no LibreOffice e salvos (com utf-8 e separador ";")

2020:
- 2020_06_producao_mar.csv: header extra inserido
    - Uma linha do header foi inserida em duplicidade no arquivo e não foi apagada.
    - Além disso, tem uma linha de totalização para algumas colunas, já eliminada

Diversos:
- Em alguns arquivos existe, antes dos dados, linhas (que chamo de "assinatura") identificando a agência, sistema, data, etc. Tais dados foram limpos de forma automática, considerando a primeira linha útil é a que se inicia por "Estado". Ainda assim, com o problema detectado no arquivo anterior, isso foi "quebrado"

Todos os casos citados acima foram tratados, de forma automática e manual, e gerou o conjunto de dados limpos com os quais passei a trabalhar. Este projeto irá "herdar" os arquivos históricos já limpos e tratará apenas das alterações necessárias nos arquivos de 2022 em diante.

Por fim, cabe incluir que já no primeiro mês de 2022, foi identificada uma inconsistência no nome do arquivo ``2022_01_01_producao.zip``, incorretamente nomeado como ``2022_01_producao-zip.zip``. Neste ponto observei a impossibilidade de manter a unificação com a série histórica e criei este novo projeto, dado que esta série não precisa passar por processos de limpeza que só podem ser feitos manualmente.