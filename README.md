# Introdução
A Assistência Farmacêutica é um dos pilares do Sistema Único de Saúde (SUS) e tem como objetivo garantir o acesso aos medicamentos de forma segura, eficaz e com qualidade para toda a população. Para isso, o SUS conta com três Componentes da Assistência Farmacêutica que visam atender às diferentes especificidades dos níveis de atenção. Dentre esses, o Componente Especializado da Assistência Farmacêutica (CEAF) é responsável por fornecer medicamentos de alto custo para pacientes que possuem doenças crônicas ou raras.  
No entanto, o acesso aos medicamentos do CEAF pode ser um desafio para os pacientes, principalmente devido à demora que pode ocorrer no processo de solicitação dos mesmos. Essa demora pode ser causada por diversos fatores, como a falta de estoque nas farmácias de dispensação, a burocracia para a autorização do fornecimento do medicamento e quantidade insuficiente de profissionais para avaliar e autorizar as solicitações. Diferentemente dos demais Componentes, o CEAF possui um fluxo mais burocrático para acesso aos medicamentos, onde o paciente necessita abrir um processo administrativo para solicitar os medicamentos. Após isso, o processo passa por mais duas instâncias, demoninadas de avaliação e autorização, antes do medicamento ser dispensado ao paciente.  
O impacto desse fluxo administrativo no CEAF pode ser grave para os pacientes, que muitas vezes dependem desses medicamentos para sobreviver ou melhorar a qualidade de vida. A falta do medicamento pode causar a interrupção do tratamento e o agravamento da doença, além de aumentar os custos com internações hospitalares e consultas médicas.  

# Objetivo
Comparar a performance de algoritmos de machine learning para predição do tempo necessário para acesso aos medicamentos do Componente Especializado da Assistência Farmacêutica (CEAF) para o estado de Santa Catarina.

# Aspectos metodológicos
## Seleção dos pacientes
Foram selecionadas apenas as solicitações de pacientes em início de tratamento. Considerou-se pacientes novos aqueles que não possuiam APAC com quantidade aprovada para o referente CID-10 ou procedimento da APAC nos últimos nove meses.   
## Coleta dos dados
A coleta de dados foi realizada em 10 de abril de 2023 e foram utilizados os registros de APAC provenientes do sistema Sistema de Informações Ambulatoriais do SUS (SIA/SUS). Foram selecionadas as APAC que com a data de solicitação entre janeiro a dezembro de 2022, no estado de Santa Catarina.  
Para a obtenção dos dados no SIA/SUS foi utilizado um programa disponibilizado no GitHub (https://github.com/ricardoronsoni/importar-apac-ceaf) que sistematiza o download dos dados do FTP do Datasus e persiste os mesmos em um banco de dados relacional.  
A partir do banco de dados criado pelo software foi realizada uma extração com os seguintes comandos SQL (Structured Query Language).
Criar tabela com os dados de Santa Catarina:     
```
create table apac.solicitacao_sc as 
select
	procedimento,
	cid,
	cns_paciente, 
	idade_paciente,
	sexo_paciente,
	municipio_residencia_paciente, 
	cnes_solicitante, 
	data_solicitacao, 
	data_autorizacao,
	competencia_dispensacao,
	data_autorizacao - data_solicitacao tempo_autorizacao
from apac.medicamento m
where 
	data_solicitacao between to_date('2021-03-01', 'YYYY-MM-DD') and to_date('2022-12-31', 'YYYY-MM-DD')
	and quantidade_aprovada > 0
	and sigla_uf_dispensacao = 'SC';
```
Adicionar campo para marcar o paciente como sendo novo no CEAF:   
```
alter table apac.solicitacao_sc add column paciente_novo bool;
```
Criar index para deixar o update mais performático:   
```
create index index_multicolumn on apac.solicitacao_sc (cns_paciente, cid, procedimento, data_solicitacao);
```
Atualizar a coluna paciente_novo com o status do registro:   
```
update apac.solicitacao_sc m set
	paciente_novo = true
	where (
			select count(*)
			from apac.solicitacao_sc m2
			where 
				m2.cns_paciente = m.cns_paciente
				and m2.cid = m.cid
				and m2.procedimento = m.procedimento
				and m2.data_solicitacao >= m.data_solicitacao - interval '9 months' and m2.data_solicitacao < m.data_solicitacao
			) = 0 
```
Realizar a extração:   
```
select *
from apac.solicitacao_sc ss 
where 
	ss.paciente_novo = true 
	and data_solicitacao >= to_date('2022-01-01', 'YYYY-MM-DD') 
```
A consulta acima resultou em um arquivo CSV contendo 117.356 registros. 
 
## Seleção das variáveis
Foi realizada uma avaliação no rol de dados disponíveis no SIA/SUS para identificar as variáveis que poderiam ser incluídas no estudo. Após a avaliação as seguintes variáveis foram selecionadas:
- procedimento  
- cid  
- idade_paciente  
- sexo_paciente  
- municipio_residencia_paciente  
- cnes_solicitante  
- data_solicitacao  
- data_autorizacao  
- competencia_dispensacao  

# Pré-processamento e Resultados
Todas as informações sobre o pré-processamento e resultados obtidos estão descritos no arquivo [Jupyter Notebook](tempo_acesso.ipynb) 