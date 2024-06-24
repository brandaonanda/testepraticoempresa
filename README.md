#Fernanda Brandão
#código feito no VSCode com Jupyter 
#Utilizado Python, Streamlit, Pandas, Ploty.express e Numpy
# As tabelas do documento foram salvas em arquivos de planilhas separadas em .csv cada
#Das tabelas salvas, foram retirados os títulos

# streamlit - criar aplicativo web interativo
import streamlit as st
# pandas - importar dados
import pandas as pd 
# plotly express - utilizar gráficos e formatar
import plotly.express as px
# numpy para numéricos
import numpy as np
#datetime para trabalhar com formato de data e hora
import datetime as dt
#calendar para trabalhar com anos, meses e anos
import calendar

#configurando o layout da página
st.set_page_config(layout="wide")

#adicionar título no topo da página
st.markdown("<h1 style='text-align: center; color: #87CEEB; font-weight: bold;'>Tratamento, geração de código e análise de dados - Globo (Fernanda Brandão)</h1>", unsafe_allow_html=True)

#df1 = dataframe de conteudo.csv com delimitador ';' aplicando a norma de padronização ISO-8859-1
df1 = pd.read_csv('conteudo.csv', delimiter=';', encoding= "ISO-8859-1")

#df2 = dataframe de consumo.csv com delimitador ';' aplicando a norma de padronização ISO-8859-1
df2 = pd.read_csv('consumo.csv', delimiter=';', encoding= "ISO-8859-1")

#mesclar dados das duas tabelas como outer
df3 = pd.merge(df1, df2, on="id_conteudo", how="outer")

#verificar dados NaN
print(df3.info())

#tratar dados NaN
df3["conteudo"]=df3["conteudo"].fillna("Não cadastrado")

df3["categoria"]=df3["categoria"].fillna("Não cadastrado")

#converter objeto data como formato de data no formato dd/mm/aaaa
df3["data"] = pd.to_datetime(df3["data"], format='%d/%m/%Y')

#converter horas/minutos (decimal) em formato de hora:minuto
def decimal_para_hhmm(horas_decimais):
    hours = int(horas_decimais)
    minutes = (horas_decimais - hours) * 60
    return f"{hours:02d}:{int(minutes):02d}"

#converter valores da coluna para string e substituir vírgula por ponto antes de converter para float
df3['horas_consumidas'] = df3['horas_consumidas'].astype(str).str.replace(',', '.').astype(float)

#aplicando a função na coluna do DataFrame
df3['horas_consumidas_hhmm'] = df3['horas_consumidas'].apply(decimal_para_hhmm)

#adicionar uma coluna com o mês em português
df3['mes'] = df3['data'].dt.month
df3['mes'] = df3['mes'].apply(lambda x: calendar.month_name[x])
df3['mes'] = df3['mes'].replace({
    'January': 'Janeiro', 'February': 'Fevereiro', 'March': 'Março', 'April': 'Abril', 'May': 'Maio', 'June': 'Junho',
    'July': 'Julho', 'August': 'Agosto', 'September': 'Setembro', 'October': 'Outubro', 'November': 'Novembro', 'December': 'Dezembro'
})

#agrupar dados por categoria e mês para calcular o total de horas consumidas e o número de plays
categoria_mes_grouped = df3.groupby(['categoria', 'mes']).agg(
    total_horas_consumidas=pd.NamedAgg(column='horas_consumidas', aggfunc='sum'),
    numero_de_plays=pd.NamedAgg(column='id_conteudo', aggfunc='count')
).reset_index()

#determinar a categoria mais assistida em cada mês
categoria_mais_assistida = categoria_mes_grouped.loc[categoria_mes_grouped.groupby('mes')['numero_de_plays'].idxmax()]

#agrupar dados por categoria para calcular o total de horas consumidas e o número de plays
categoria_grouped = df3.groupby('categoria').agg(
    total_horas_consumidas=pd.NamedAgg(column='horas_consumidas', aggfunc='sum'),
    numero_de_plays=pd.NamedAgg(column='id_conteudo', aggfunc='count')
).reset_index()

#sidebar para seleção de categoria
categorias = ['Todas as Categorias'] + categoria_grouped['categoria'].unique().tolist()
categoria_selecionada = st.sidebar.selectbox('Selecione uma Categoria', categorias)

#filtrar os dados pela categoria selecionada
if categoria_selecionada == 'Todas as Categorias':
    dados_filtrados = categoria_grouped
else:
    dados_filtrados = categoria_grouped[categoria_grouped['categoria'] == categoria_selecionada]

#agrupar dados por conteudo para calcular o número de plays por mês
conteudo_grouped = df3.groupby(['conteudo', 'mes']).agg(
    numero_de_plays=pd.NamedAgg(column='id_conteudo', aggfunc='count')
).reset_index()

#sidebar para seleção de conteudo
conteudos = ['Todos os Conteúdos'] + df3['conteudo'].unique().tolist()
conteudo_selecionado = st.sidebar.selectbox('Selecione um Conteúdo', conteudos)

# **_Filtrar os dados pelo conteudo selecionado_**
if conteudo_selecionado == 'Todos os Conteúdos':
    dados_filtrados_conteudo = conteudo_grouped
else:
    dados_filtrados_conteudo = conteudo_grouped[conteudo_grouped['conteudo'] == conteudo_selecionado]

#agrupar dados por usuário para calcular o número de plays por conteúdo
usuario_grouped = df3.groupby(['id_user', 'conteudo']).agg(
    total_horas_consumidas=pd.NamedAgg(column='horas_consumidas', aggfunc='sum')
).reset_index()

#criando as colunas da dashboard
col1, col2 = st.columns(2)
col3, col4 = st.columns(2)
col5, col6 = st.columns(2)

#criar gráficos
fig_horas = px.bar(dados_filtrados, x='categoria', y='total_horas_consumidas',
                   title='Total de Horas Consumidas por Categoria',
                   labels={'total_horas_consumidas': 'Total de Horas Consumidas', 'categoria': 'Categoria'})

fig_plays = px.bar(dados_filtrados, x='categoria', y='numero_de_plays',
                   title='Número de Plays por Categoria',
                   labels={'numero_de_plays': 'Número de Plays', 'categoria': 'Categoria'})

#exibir os gráficos
col1.plotly_chart(fig_horas, use_container_width=True)
col2.plotly_chart(fig_plays, use_container_width=True)

#criar gráfico de conteudo mais assistido por mês
fig_conteudo_mais_assistido = px.bar(dados_filtrados_conteudo, x='mes', y='numero_de_plays', color='conteudo',
                                     title='Conteúdo Mais Assistido por Mês',
                                     labels={'mes': 'Mês', 'numero_de_plays': 'Número de Plays', 'conteudo': 'Conteúdo'})

#exibir o gráfico de categoria mais assistida por mês
col3.plotly_chart(fig_conteudo_mais_assistido, use_container_width=True)

#mostrar tabela com colunas "conteudo" e "categoria" sem índice e com títulos substituídos
tabela_conteudo_categoria = df3[['id_conteudo','conteudo', 'categoria']].drop_duplicates().rename(columns={'id_conteudo': 'ID do Conteúdo','conteudo': 'Conteúdo', 'categoria': 'Categoria'})

#adicionar título e exibir tabela
col4.markdown("**Referência de categorias**")
col4.write(tabela_conteudo_categoria.to_html(index=False), unsafe_allow_html=True)

#mostrar tabela de minutos por play por usuário
minutos_por_play = df3.groupby('id_user').agg(
    total_minutos_consumidos=pd.NamedAgg(column='horas_consumidas', aggfunc=lambda x: np.sum(x) * 60)
).reset_index()

#adicionar título e exibir tabela
col5.markdown("**Minutos por play por usuário**")
col5.write(minutos_por_play[['id_user', 'total_minutos_consumidos']].rename(columns={'id_user': 'Usuário', 'total_minutos_consumidos': 'Minutos Consumidos'}).to_html(index=False, escape=False), unsafe_allow_html=True)

#criar gráfico de consumo por categoria
fig_consumo_categoria = px.bar(dados_filtrados, x='categoria', y='total_horas_consumidas',
                               title='Consumo por Categoria',
                               labels={'total_horas_consumidas': 'Horas Consumidas', 'categoria': 'Categoria'})

#exibir o gráfico de consumo por categoria
col6.plotly_chart(fig_consumo_categoria, use_container_width=True)
