
# Funções para consultas automáticas da API do Projeto Tá de Pé em Python

A seguir, apresento funções para a consulta da API do Tá de Pé e para armazenar os dados em um dataframe.

## Obtendo autorização

A API do TDP exige um token de autorização. Você obtem esse token na documentação da API do TDP. Na documentação vá em Autorização >> Obter token e clique em Auths. Veja se o menu esquerdo está como Console e vá em Call Resource. Desça a tela até Response Body e pegue o código que aparece depois de token, que será algo como "Bearer yy...". Copie esse token, ele será necessário para executar os comandos.

## Setando URL

A URL é o tipo de consulta que queremos fazer. Para mais informações sobre o que é cada uma das consultas, acesse a documentação da API em https://tadepe.docs.apiary.io/ .


```python
import requests
import json
import pandas as pd
from pandas.io.json import json_normalize
```


```python
# Obter obras:
obter_obras = 'http://tadepe.transparencia.org.br/api/projects/content?page='

# Obter contatos:
obter_contatos = 'http://tadepe.transparencia.org.br/api/contacts/content?page='

# Obter alertas:
obter_alertas = 'http://tadepe.transparencia.org.br/api/inspections/content?page='

# Obter evidências:
obter_evidencias = 'http://tadepe.transparencia.org.br/api/evidences/content?page='

# Obter instituições:
obter_instituicoes = 'http://tadepe.transparencia.org.br/api/institutions/content?page='

# Obter mensagens:
obter_mensagens = 'http://tadepe.transparencia.org.br/api/messages/content?page='

# Obter respostas:
obter_respostas = 'http://tadepe.transparencia.org.br/api/answers/content?page='
```

## Função para obter todas as respostas de uma requisição:


```python
def fetch_api(url, token, pg = 1):
    nextpage = True
    headers = {"Authorization": token}
    a = list()
    while nextpage:
        base_url = url + str(pg)
        r = requests.get(base_url, headers=headers).json()
        b = r.get('data')
        a = a + b
        nextpage = r.get('links').get('next')
        pg = pg + 1
    
    return a
```
O parâmetro "pg" refere-se a página que a função iniciará o armazenamento. 

Exemplo de uso:


```python
token = 'token obtido'

#Exemplo 1:
lista_result = fetch_api(url = obter_alertas, token = token)

#Exemplo 2:
lista_result = fetch_api(url = obter_alertas, token = token, pg=30)
```

## Transformando a lista resultante em um dataframe:


```python
def result_to_df(lista):
    c = lista[1]['attributes'].keys()
    df_int = pd.DataFrame(columns = c)
    for elemento in range(0,len(lista)):
        y = lista_result[elemento]['id']
        x = lista[elemento]['attributes']
        x = pd.DataFrame(x, index=[0])
        x['inspection_id'] = y
        df_int = df_int.append(x)
    return df_int
```

Exemplo de uso:


```python
df = result_to_df(lista = lista_result)
```

## Alertas válidos e com problemas

No app "Tá de Pé" são enviados muitos alertas por cidadãos que na realidade estão apenas realizando testes, e não são fotos de obras. Além disso, recebemos dois tipos de alertas: aqueles que precisam de verificação de um engenheiro para atestar o atraso e os que não precisam, pois já possuem atraso em relação à data de entrega ou possuem outras incongruências (ausência de placa, endereço incorreto, etc.)

Para simplificar o processo de leitura dos dados, utilize a função abaixo para obter um dataframe apenas com os alertas válidos, já descartando aqueles que foram rejeitados (não possuiam atraso) ou descartados (não se tratava de um alerta verdadeiro):

```python
def alertas_validos(df):
    df['valid'] = np.where((~df['status-incongruity'].isna()) |
                           ((df['status'] != 'discarded')  & 
                            (df['status'] != 'rejected')), 1, 0)
    df['type-alert'] = np.where((~df['status-incongruity'].isna()), 'incongruity_based', 'delay_based')
    df['status-incongruity'] = df['status-incongruity'].str.replace("incongruity_", "", regex = False) 
    df['status2'] = np.where((~df['status-incongruity'].isna()), 
                             df['status-incongruity'], df['status'])
    df[['created-at','updated-at']] = df[['created-at','updated-at']].apply(pd.to_datetime, format="%Y-%m-%d %H:%M:%S")
    df = (df.query('valid == 1')
            .query('status2 != "pending"')
            .drop(columns= ['status', 'status-incongruity', 'valid'])
            .rename(columns={'status2' : 'status'}))
    
    return df
```

Exemplo:

```python
#Obtendo a lista dos alertas
lista_result = fetch_api(url = obter_alertas, token = token)

#Transformando alertas em df
df = result_to_df(lista = lista_result)

#Guardando apenas aqueles que já foram validados
alertas_final = alertas_validos(df = df)

```
