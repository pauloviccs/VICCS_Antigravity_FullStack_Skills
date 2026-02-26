---
description: Análise de arquitetura e implementação de um Portfólio da Galeria do The Sims 4, sincronizando dados da EA via API, exibindo sem downloads, em design Liquid Glass.
---

# Sims 4 Gallery Portfolio Architecture & Implementation

Este guia documenta a arquitetura, base de dados e experiência do usuário (UX) necessárias para replicar e integrar a "Galeria Oficial do The Sims 4" (EA Games) dentro de um projeto SaaS ou website pessoal.

O escopo é focado em exibição e portfólio de um usuário criador de conteúdo, utilizando sincronização inteligente de dados e um design system premium no formato **Liquid Glass**.

---

## 1. O Problema e o Fluxo Norteador ("North Star Utility")

O objetivo do sistema **não é ser um repositório para contornar o jogo** baixando arquivos (`.trayitem`), mas sim ser uma **vitrine premium e curada** para exibir do que o criador é capaz.

A solução resolve o problema de descentralização e apresentação, garantindo ao usuário a possibilidade de integrar seu EA ID ao seu próprio app/SaaS.

**Fluxo Base (Mental Model):**

1. O criador de conteúdo acessa o painel de Configurações do seu site/SaaS.
2. Adiciona o seu **Nickname/EA ID**.
3. Consome a API da EA para buscar o catálogo.
4. Escolhe um limite máximo de **50 lotes/sims** para compor a sua "Galeria Pessoal" (para otimizar Storage, Cache e Requisições Trocadas - *Egress*).
5. Visitantes visualizam esses cards e clicam para ler todos os detalhes originais da página da EA, buscando a obra por conta própria no jogo original usando o nome/hashtag.

---

## 2. A "Data Source of Truth" (Extração de Dados da EA)

A página da EA funciona inteiramente como uma SPA (Single Page Application) e consome endpoints HTTP públicos, porém sem documentação oficial.

Para sua aplicação, o backend será o responsável pelas integrações:

### O Algoritmo de Sincronização

- **Search pelo Usuário:**
  Na ação de vinculação, seu Backend (Node/Python/etc) fará requisições para `https://gallery.services.ea.com/gallery/api/v1/` pesquisando pelo autor exato (*Search by EA Account ID*).
- **Lista de Assets:**
  A API trará a "Lista de itens", exibindo Nome e a Imagem de Preview. O usuário fará a curadoria dos 50 itens desejados com base nisso.
- **Deep Fetch & Reverse Engineering:**
  Para os dejesados, o backend vai buscar o detalhamento completo pelo ID do item da galeria, salvando no seu banco (PostgreSQL/Supabase):
  - Thumbnail Original (Imagem grande e secundárias)
  - Estatísticas de Likes e Downloads do EA original
  - Comentários e Descrição Textual da peça original
  - Pacotes Necessários (Expansões, Pacotes de Jogo e Coleções de Objetos)

*(Atenção aos Rate Limits: O tráfego direto do cliente para a EA pode resultar em falhas de CORS; o ideal é que esse processo aconteça no Server-Side, protegendo endpoints de bloqueios).*

---

## 3. Estrutura Proposta de Banco de Dados

Esta etapa garante a otimização citada no requisito #5 (pouco impacto em Storage e consultas ultra-rápidas locais).

```sql
-- Ligação entre usuário do seu sistema -> Conta da EA (Adicionado nas Configurações)
CREATE TABLE user_ea_profiles (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id),
  ea_username VARCHAR(100) NOT NULL,
  last_synced_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Tabela da Galeria Pessoal (Limitado a 50 via regras de negócio no Back-end/Trigger)
CREATE TABLE user_gallery_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES user_ea_profiles(user_id),
  
  -- ID e Dados Básicos da EA para referência e link original
  ea_item_id VARCHAR(100) UNIQUE NOT NULL, 
  title VARCHAR(255) NOT NULL,
  description TEXT,
  
  -- Metadados Numéricos (Downloads da EA, Likes, Comentários)
  metrics JSONB DEFAULT '{}',
  
  -- Array ou JSON da lista de pacotes The Sims 4 necessários
  required_packs JSONB DEFAULT '[]', 
  
  -- Snapshot de comentários iniciais originais
  original_comments JSONB DEFAULT '[]', 
  
  -- Imagens. É recomendável hospedar um cache das imagens no seu Storage 
  -- para evitar que Links da CDN da EA quebrem no futuro.
  thumbnail_url TEXT, 
  extra_images_urls TEXT[], 

  -- Customização e Ordenação do Portfólio
  is_pinned BOOLEAN DEFAULT false,
  sort_order INT DEFAULT 0,
  
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 4. Design & Interaction Specifications ("Visual Hierarchy")

### A Pegada Visão ("Liquid Glass")

O sistema deve exalar uma interface refinada: fundos translucentes (efeito de vidro esfumaçado), micro-interações fluidas e tipografia suave, nunca sobrepondo as imagens - que no The Sims 4 já são densas.

- **Background:** Pode conter leves `radial-gradients` ou 'Mica/Acrylic' da paleta do site pai.
- **Cards Principais (3-Second Rule):**
  - Foco central na miniatura grande da construção/sim.
  - Overlay simples: Apenas Nome.
  - O usuário entende instantaneamente que se trata de uma Galeria Curada no momento em que a página carrega.
- **Transição ao Clicar (Interaction Cost):**
  - **Não há páginas novas para cada lote**. A galeria não recarrega.
  - Framer Motion `LayoutGroup` para expandir suavemente o card (Hero Modal/Drawer), revelando todos os detalhes contidos no JSON.
- **Conteúdo Exibido no "Inside Card":**
  1. A foto em escala maior e carrossel de fotos secundárias.
  2. Ícones de pacotes renderizados debaixo de cards "Glass" menores (DLC Needed).
  3. Comentários originais puxados da EA mostrados integralmente.
  4. Nome original e Foto da Casa em destaque.
- **Navegação (Future-proofing):**
  - Com o teto de 50 itens, o layout fará o fetch inicial dos 50 de vez.
  - A interface utilizará paginação em formato **X-Slides/Carousels** suaves ou numeração minimalista embaixo de blocos de 10-12 itens, mantendo a leveza ("no oversaturation") da página.
- **Ações:** Zero botão de "Baixar". Apenas instrução clara ou hastag em destaque para que seja pesquisada ingame.

---

## 5. Protocolo Adicional de Escalabilidade & Performance

- **Tamanho das imagens:** As Thumbnails puxadas da EA podem ser guardadas no bucket local com o formato comprimido `.webp`, mantendo uma galeria limpa, sem estourar quotas de Egress caso seu projeto suba para centenas de usuários criadores de conteúdo.
- **Cache do Backend:** Seus usuários não devem atualizar a página exigindo 50 consultas de detalhes nos servidores da EA ao mesmo tempo. A vinculação via "Página de Configurações" serve exatamente para trazer a carga da EA 1 única vez para seu modelo local de dados, refletindo com latência minúscula para os visitantes finais.
