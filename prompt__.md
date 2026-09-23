[CONTEXT]
Preciso de uma aplicação web em Python com Streamlit para ser usada por estudantes em sala de aula.
Eles NÃO devem mexer no código ou na arquitetura. Eles usarão apenas a interface gráfica para consultar dados de unidades de um site.
A aplicação deve fazer o scraping/RAG em segundo plano do site "https://gratuitos.netlify.app/" e responder dúvidas do usuário (como endereços de unidades) usando a API da azure Foundry.

[OBJECTIVE]
Gerar o código completo e executável do arquivo `app.py`. A aplicação deve:

1. Ter uma interface extremamente simples, limpa e intuitiva para o aluno.
2. Ocultar totalmente chaves de API, scripts de scraping e detalhes técnicos.
3. Conter um campo de texto para a dúvida e um botão visível "Buscar Unidade / Perguntar".
4. Fazer o scraping automático do site "https://gratuitos.netlify.app/" em cache (usando @st.cache_data) para não travar a navegação.
5. Enviar a pergunta do aluno junto com o contexto raspado para a API da azure e exibir a resposta de forma clara na tela.

[STYLE]
Código Python production-ready, totalmente funcional, usando `streamlit`, `azure`, `requests` e `beautifulsoup4`.

[TONE]
Direto, instrutivo e limpo.

[AUDIENCE]
Professor que vai rodar o servidor localmente (ou na nuvem) e disponibilizar apenas o link final do navegador para os estudantes usarem.

[RESPONSE FORMAT]
Entregue a resposta no seguinte formato:

1. Lista de comandos `pip install` necessários.
2. Código Python completo do `app.py` com suporte a `st.secrets` para a chave `AZURE FROUNDRY API AGENT`.
3. Instruções curtas de como o professor deve rodar o projeto.
Sem textos teóricos ou introduções desnecessárias.

reutilize esse código de forma que, 
apenas inclua o webscrapping  no rag e mantenha os nomes da variáveis do ambiente. 
insira o streamlit e o beutifuoul soap no rag

Não altere o nome das chaves de api do aquivo  .env 

import os
from dotenv import load_dotenv
from openai import AzureOpenAI, APIError
import sqlite3
import pandas as pd

dados_csv = pd.read_csv('dados.csv')
T = pd.read_csv('tic.csv')

con = sqlite3.connect('banco.db')
cursor = con.cursor()

cursor.execute('''CREATE TABLE IF NOT EXISTS vendas(
                 venda REAL,
                 vendedor TEXT,
                 id TEXT
               )
''')

con.commit()

cursor.execute('INSERT INTO vendas VALUES(?,?,?)', (1000,'Thiago',1))
cursor.execute('INSERT INTO vendas VALUES(?,?,?)', (10000,'Maria',2))
cursor.execute('INSERT INTO vendas VALUES(?,?,?)', (1220,'Fernando',3))

# crud
con.commit()

cursor.execute('SELECT * FROM vendas')
dados = cursor.fetchall()
contexto = f'{dados[0]}, {dados[1]}, {dados[2]}'

load_dotenv(override=True)

client = AzureOpenAI(
    azure_endpoint=os.getenv("ENDPOINT"),
    api_key=os.getenv("API_KEY"),
    api_version=os.getenv("API_VERSION", "2025-04-01-preview")
)

deployment_name = os.getenv("GPT5_MODEL")


def validar_configuracao():
    faltando = []
    if not os.getenv("ENDPOINT"):
        faltando.append("ENDPOINT")
    if not os.getenv("API_KEY"):
        faltando.append("API_KEY")
    if not deployment_name:
        faltando.append("GPT5_MODEL")

    if faltando:
        print("[Erro de configuração] As seguintes variáveis não foram definidas no .env:")
        for var in faltando:
            print(f"  - {var}")
        print("Preencha o arquivo .env e tente novamente.")
        return False
    return True


def main():
    if not validar_configuracao():
        return

    print("inciando ... ")

    while True:
        pergunta = input("Digite sua pergunta ou sair: ").strip()

        if pergunta.lower() == "sair":
            print("Atendimento finalizado. Até logo!")
            break

        if not pergunta:
            print("Por favor, digite uma pergunta válida.")
            continue

     
       # recuperação de busca
        resultados_busca = T[T.apply(lambda row: row.astype(str).str.contains(pergunta, case=False).any(), axis=1)]
       
   
        if not resultados_busca.empty:
            dados_recuperados = resultados_busca.head(5).to_string(index=False)
        else:
            dados_recuperados = T.head(5).to_string(index=False)

       
        prompt = f"""Você é um historiador especialista em titanic, use apenas os dados que vem do dataset abaixo para responder aos usuários de forma que esteja ensinando sempre:

       
da
{dados_recuperados}


"""
 

        try:
            response = client.chat.completions.create(
                model=deployment_name,
                messages=[
                    {
                        "role": "system",
                        "content": prompt, # Passando o prompt com o RAG dinâmico
                    },
                    {
                        "role": "user",
                        "content": pergunta
                    }
                ]
            )

            resposta_texto = response.choices[0].message.content
            print(f"Resposta:{resposta_texto}")

        except APIError as e:
            print(f"[Erro na API]: {e}")
        except Exception as e:
            print(f"[Erro inesperado]: {e}")


if __name__ == "__main__":
    main()