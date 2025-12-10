# Projeto de Banco de Dados – Redoar
integrantes:
Alexciane Lima ;
Fabiana Amorim ;
Juan Sá Barreto

## 1. Descrição do Modelo Lógico (redoar_db)

O banco de dados redoar_db foi modelado para suportar os principais fluxos de negócio do aplicativo Redoar: cadastro de usuários (doadores, receptores e ONGs), publicação e moderação de itens, agendamento de coletas, registro de doações efetivas e sistema de pontuação gamificada. O esquema foi implementado em MySQL utilizando utf8mb4_unicode_ci, adequado para textos em português. [file:37]

---

## 2. Tabelas do Modelo

### 2.1 Tabela USUARIO

A tabela USUARIO armazena todos os usuários da plataforma, sejam doadores individuais, empresas doadoras, receptores individuais ou organizações sociais (ONGs). [file:37]

Principais campos e regras:

- id_usuario: chave primária numérica auto‑incremento.
- nome, email, senha_hash: campos obrigatórios; email é único e indexado.
- tipo: ENUM com os perfis doador_ind, doador_emp, receptor_ind, ong.
- cpf_cnpj: identificação fiscal (PF ou PJ), com unicidade.
- endereco_logradouro, endereco_cidade, endereco_estado, endereco_cep: endereço principal do usuário.
- data_cadastro, pontuacao_total, conta_ativa: controlam cadastro, pontuação acumulada e status da conta.

---

### 2.2 Tabela ONG (especialização de USUARIO)

A tabela ONG representa a especialização dos usuários cujo tipo é organização social, armazenando informações específicas de entidades intermediárias que gerenciam parte das doações. [file:37]

- id_ong: chave primária e, ao mesmo tempo, chave estrangeira para USUARIO(id_usuario) (herança/especialização).
- razao_social, nome_fantasia, area_atuacao: dados institucionais da ONG.
- capacidade_gestao: métrica de capacidade de processamento de doações.
- ON DELETE CASCADE: remoção de um usuário ONG apaga o registro correspondente em ONG.

---

### 2.3 Tabela ITEM

A tabela ITEM registra todos os bens cadastrados para doação na plataforma Redoar. [file:37]

- id_item: chave primária auto‑incremento.
- titulo, descricao: informações descritivas do produto.
- categoria: ENUM com tipos de produto.
- condicao: ENUM com estado de conservação (novo, seminovo, usado etc.).
- status: acompanha o ciclo de vida do item (rascunho, pendente de moderação, aprovado, disponível, reservado, em coleta, doado, rejeitado, expirado).
- id_doador: FK obrigatória para USUARIO(id_usuario).
- id_ong_responsavel: FK opcional para ONG(id_ong).
- Datas de criação, moderação e expiração, além de motivo_rejeicao.
- Índices: idx_status, idx_categoria, idx_doador.

---

### 2.4 Tabela FOTO_ITEM

A tabela FOTO_ITEM armazena as imagens associadas a cada item, permitindo múltiplas fotos por anúncio. [file:37]

- id_foto: chave primária auto‑incremento.
- id_item: FK obrigatória para ITEM(id_item) com ON DELETE CASCADE.
- url_foto: caminho da imagem.
- ordem: ordem de exibição.
- legenda: descrição opcional.
- Índice: idx_item para consultas por item.

---

### 2.5 Tabela MODERACAO

A tabela MODERACAO registra o histórico de análise e curadoria dos itens. [file:37]

- id_moderacao: chave primária.
- id_item: FK para ITEM(id_item).
- id_moderador: FK opcional para USUARIO(id_usuario).
- acao: ENUM (aprovado, rejeitado, pendente_ajuste).
- observacoes: comentários e justificativas.
- data_moderacao: data/hora da decisão, indexada para relatórios.

---

### 2.6 Tabela AGENDAMENTO_COLETA

A tabela AGENDAMENTO_COLETA modela o fluxo de combinação de data e local para retirada dos itens. [file:37]

- id_agendamento: chave primária.
- id_item: FK única para ITEM(id_item) (um agendamento por item).
- id_receptor: FK para USUARIO(id_usuario) (quem recebe).
- id_coletor: FK para USUARIO(id_usuario) (quem coleta, se houver).
- Datas de solicitação e de coleta agendada, horario_preferencial e endereco_coleta.
- status: ENUM (pendente, confirmado, em_rota, coletado, cancelado, falha).
- observacoes e índices por data agendada e status.

---

### 2.7 Tabela DOACAO

A tabela DOACAO consolida o registro final da doação, vinculando item, doador, receptor e ONG intermediária. [file:37]

- id_doacao: chave primária.
- id_item: FK única para ITEM(id_item) (um item → uma doação efetiva).
- id_doador, id_receptor_final: FKs para USUARIO(id_usuario).
- id_ong_intermediaria: FK para ONG(id_ong).
- data_efetivacao: data de conclusão da doação.
- status_receptor: ENUM (recebido, utilizado, distribuido).
- feedback_receptor: comentário livre sobre a experiência.
- avaliacao_doador: nota de 1 a 5 (CHECK), usada para reputação e pontos.

---

### 2.8 Tabela PONTUACAO e Trigger de Gamificação

A tabela PONTUACAO implementa o sistema de gamificação, registrando conquistas e pontos atribuídos a cada usuário. [file:37]

- id_pontuacao: chave primária.
- id_usuario: FK para USUARIO(id_usuario).
- id_doacao: FK para DOACAO(id_doacao).
- pontos: quantidade de pontos concedidos.
- tipo_pontuacao: ENUM (doacao, avaliacao_positiva, rapidez, item_novo).
- data_conquista: data/hora da pontuação.

A trigger atualiza_pontuacao_total, executada após INSERT em PONTUACAO, atualiza automaticamente o campo pontuacao_total de USUARIO, mantendo o saldo consolidado. [file:37]

---

## 3. Visão view_doacoes_ong

A view view_doacoes_ong apoia relatórios gerenciais sobre desempenho e impacto das ONGs parceiras. [file:37]

- Agrega as doações por ONG, exibindo:
  - Nome fantasia da ONG.
  - Total de doações.
  - Quantidade de itens novos.
  - Média da avaliação atribuída pelos doadores.
- Sintetiza indicadores alinhados ao ODS 12 (Consumo e Produção Responsáveis), permitindo monitorar contribuição das ONGs na redução do desperdício e na redistribuição solidária de bens. [file:37]

---

## 4. Mapeamento Conceitual → Lógico

| Entidade conceitual      | Tabela física      | Chave primária     | Principais relacionamentos                                                  |
|--------------------------|--------------------|--------------------|-----------------------------------------------------------------------------|
| Usuário                  | USUARIO          | id_usuario       | Referenciado por ONG, ITEM, MODERACAO, AGENDAMENTO_COLETA, DOACAO, PONTUACAO |
| ONG                      | ONG              | id_ong           | FK para USUARIO(id_usuario); referenciada por ITEM e DOACAO           |
| Item                     | ITEM             | id_item          | FK para USUARIO (id_doador) e ONG (id_ong_responsavel); base para FOTO_ITEM, MODERACAO, AGENDAMENTO_COLETA, DOACAO |
| Foto de Item             | FOTO_ITEM        | id_foto          | FK para ITEM(id_item)                                                     |
| Moderação de Item        | MODERACAO        | id_moderacao     | FKs para ITEM(id_item) e USUARIO(id_usuario)                            |
| Agendamento de Coleta    | AGENDAMENTO_COLETA | id_agendamento | FKs para ITEM(id_item), USUARIO(id_receptor) e USUARIO(id_coletor)   |
| Doação                   | DOACAO           | id_doacao        | FKs para ITEM, USUARIO (doador e receptor), ONG                       |
| Pontuação / Gamificação  | PONTUACAO        | id_pontuacao     | FKs para USUARIO(id_usuario) e DOACAO(id_doacao)                        |

https://www.figma.com/make/IKqqQwAqQaJwc7pSh4h233/ReDoar?node-id=0-1&p=f
