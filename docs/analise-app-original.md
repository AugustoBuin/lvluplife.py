# Level up Life — análise do app original (engenharia reversa do APK v67)

Documento de referência para o remake. Fonte: APK `com.lvluplife.lvluplife` versionCode/versionName 67
(minSdk 16, targetSdk 27), descompilado com jadx e apktool, mais os dois repositórios públicos
(`lvluplife.py`, cliente Python da API; `LevelupLifeOFI`, userstyle CSS do site).

Dados extraídos ficam em `docs/data/`:

- `achievements.json` / `achievements.csv` — catálogo completo de 567 achievements embutidos no app.
- `game-tables.json` — tabela de XP por nível, XP base por dificuldade, tabelas do achievement customizado, regras de energia e de bloqueio.

## 1. Conceito

RPG de vida real: o usuário registra "achievements" (conquistas) que fez no mundo real, ganha XP e pontos em seis
skills, sobe de nível e desbloqueia achievements mais difíceis. Funciona no sistema de honra (sem verificação).
Possui camada social: feed de atividade, amigos, comentários, ranking, notificações. Visual retrô 8-bit, tema escuro.

Slogan: *"Get motivated by tracking your everyday achievements in this real-life RPG!"*

## 2. Stack técnica do app original

| Área | Tecnologia |
|---|---|
| Plataforma | Android nativo, Java, Support Library 27 |
| Rede | Volley (JSON sobre HTTP POST/GET, form-urlencoded) |
| Persistência local | greenDAO sobre SQLite (`usertasks-db`) |
| Imagens | Picasso; EasyPhotoPicker para galeria/câmera |
| Datas | Joda-Time |
| UI | RecyclerView, ViewPager com SlidingTabLayout, DrawerLayout, RangeBar (appyvet), FloatingActionButton |
| Login social | Facebook SDK |
| Monetização | AdMob (banners + interstitial), Google Play Billing (doações removem anúncios) |
| Telemetria | Firebase Analytics, Crashlytics/Fabric |
| Fontes | Share Tech, Share Tech Mono, Press Start 2P (assets) |
| Backend | PHP em `lvluplife.com/app/v3`, `lvluplife.com/app/v5` e `api1.lvluplife.com/app/v5` |

Cores principais (tema escuro): fundos `#0c0e12`, `#12141a`, `#191c21`, `#1d2327`, `#283035`, `#364042`;
verde `#20e87b` / `#00ca60`; azul `#24d1ff` / `#22c4fa` / `#1ea1c8`; aqua `#83f0dc`; cinza `#999999`; vermelho `#c92f3a`.

## 3. Mecânica de jogo

### 3.1 Skills (atributos)

| Código | Nome | Definição oficial (texto de ajuda) |
|---|---|---|
| STR | Strength | Saúde física e fitness |
| CUL | Culture | Mundo, mente aberta, história, artes, tradições |
| ENV | Environment | Melhorar o entorno (casa, cidade, país, mundo) |
| CHA | Charisma | Habilidades sociais, interação, comunicação |
| TAL | Talent | Habilidade ou aptidão especializada |
| INT | Intellect | Conhecimento, estudo, pesquisa |

Todos começam em 0. O valor de cada skill é a soma dos pontos ganhos. As barras do perfil são relativas à maior
skill do usuário (maior skill = 100%).

### 3.2 XP e níveis

- XP total = soma de `gainedxp` dos achievements concluídos não excluídos.
- Nível = posição na tabela de 61 limiares (`level_xp_thresholds` em `game-tables.json`). Nível 2 aos 200 XP, nível 10 aos 18.675 XP, nível 20 aos 302.850 XP, nível 60 aos 43.175.485 XP. Nível máximo 60.
- Achievements desbloqueiam por nível de 1 a 20 (`lvl`). Acima do nível 20 não há novos desbloqueios.
- Barra de XP mostra `xp_min / xp_max` dentro do nível atual e porcentagem.
- Ao subir de nível aparece um diálogo "You have leveled up to LVL N" com a arte `levelup.png`; o evento também entra no feed (`activity_type = levelup`).
- O texto de ajuda promete "special perks at certain levels". No código v67 o único perk residual é a mensagem "Your maximum energy has increased! You have gained one heart!", mas o máximo de energia está fixo em 20.

### 3.3 XP e pontos de cada achievement

Cada achievement tem `points` (XP), `lvl` (nível mínimo), seis inteiros de skill e `limit` (frequência):

| limit | Significado | Bloqueio após concluir |
|---|---|---|
| 1 | Max. uma vez por dia | até 23:59:59 do dia (fuso do usuário) + 24 h |
| 2 | Max. uma vez por semana | + 168 h |
| 3 | Max. uma vez por mês | + 1 mês |
| 4 | Max. uma vez por ano | + 1 ano |
| 5 | Uma única vez ("bucket list") | permanente |

Regra verificada no catálogo: `points = base_xp[lvl] × limit`. Ex.: nível 5 base 150 → 150/300/450/600/750.
Quanto maior a frequência permitida (limit), mais XP.

### 3.4 Energia (corações)

- Máximo de 20 conclusões por janela deslizante de 60 minutos.
- O app conta conclusões não excluídas nos últimos 60 min; corações restantes = 20 − contagem.
- Ao chegar a 0: diálogo "HOURLY LIMIT REACHED — Limit of 20 achievements within 1 hour". Um timer mostra quando o coração mais antigo regenera.
- Após o usuário bater o limite mais de 20 vezes (contador local), passa a exibir interstitial de anúncio nesse momento.

### 3.5 Catálogo de achievements

567 achievements embutidos no app (também sincronizados do servidor), `tasknum` de 1 a 579.

| Categoria | Código | Qtde | Subcategorias |
|---|---|---|---|
| Reading | rea | 159 | "Top 150" (150 livros clássicos) |
| Arts & Creativity | art | 63 | Music, Writing, Photography |
| Fitness & Health | fit | 52 | Sports |
| Travel | tra | 50 | Destinations |
| Fun | fun | 35 | |
| Outdoors | out | 35 | |
| Mental | men | 33 | |
| Food & Cooking | foo | 26 | |
| Humanity | hum | 24 | |
| Household & DIY | hou | 24 | |
| Career & Finances | car | 22 | |
| School & Learning | sch | 19 | |
| Social | soc | 18 | |
| (legado) | old | 6 | não listado na UI |
| (anômalo) | new | 1 | provavelmente erro de dados |

Distribuição por nível: 39 no nível 1, caindo a ~20 por nível até o 20. Por frequência: 196 diárias, 87 semanais,
62 mensais, 30 anuais, 192 únicas. Além dessas, a tela de categorias mostra "MY GOALS" (favoritos marcados com estrela).

### 3.6 Achievements customizados

- Criados pelo usuário dentro de uma categoria, visíveis só para ele. Nome único por categoria.
- Formulário: nome, subcategoria (spinner), dificuldade (RangeBar 1..min(nível do usuário, 20)), frequência (1 a 5), seis skills com opções None/Low/Medium/High.
- XP = `base_xp[dificuldade] × limit`. Skill = `tabela[dificuldade][opção] × multiplicador[limit]` (1, 1, 1.5, 2, 3), arredondado, mínimo 1.
- Recebem `tasknum ≥ 9001` por usuário. Podem ser editados e excluídos. No modo offline são gerados localmente.

### 3.7 Ações sobre um achievement (legenda da tela de ajuda)

| Ícone | Ação |
|---|---|
| Quill (pena) | Editar descrição/foto da conclusão |
| Bomb | Excluir conclusão (perde os pontos) ou excluir custom |
| Lock / Key | Ocultar / reexibir achievement na listagem |
| Star / Star apagada | Meta atual / adicionar como meta |
| Balão | Comentários (ícones cmt0..cmt10 mostram a contagem) |
| Scroll | Ver atividade daquele achievement |

Listagem de categoria tem três visões: Default, Hidden, All. Preferência "Auto-hide achievements" oculta
automaticamente achievements bloqueados por já terem sido concluídos recentemente.

### 3.8 Conclusão de um achievement

Tela `A_EditAchievementForm`: nome, XP, skills, "Unlocked at LVL", texto de frequência, campo de descrição
("Where? When? Why? How'd it go? Who were you with?") e foto opcional (galeria/câmera, enviada em base64).
A conclusão vira um `usertask` com data/hora UTC, descrição, XP e skills ganhos, e opcionalmente uma imagem.

## 4. Telas e navegação

```
Login (A_Login / LoginFragment)
 ├─ login usuário/senha, login Facebook, criar conta (username, e-mail opcional, senha ≥ 6),
 │  esqueci a senha, Play Offline (só um nome)
 └─ MainActivity (toolbar + abas + drawer + FAB verde)
     ├─ Abas: Friends | Community | Me  (feed de atividade; offline só "My Activity")
     ├─ FAB "+" → A_CatList (grade de categorias com ícones pixel art + MY GOALS)
     │     └─ A_TaskList (lista da categoria; menu: View Hidden Tasks, + Add Custom Achievement, Settings, Log Out)
     │           ├─ A_EditAchievementForm (concluir / editar conclusão)
     │           └─ A_NewCustomForm (criar / editar custom)
     ├─ A_AchievementType (atividade de um achievement: abas Me | Friends | Community)
     ├─ A_Comments (abas Achievement | Comments; adicionar/excluir comentário)
     ├─ A_ProfileUser (abas Profile | Activity; botão add/accept/remove friend; "Report this user")
     │     └─ POP_ReportUser (motivos: inappropriate, spamming, cheating, harassment, other)
     ├─ POP_ViewImage (foto da conclusão em tela cheia)
     ├─ SettingsActivity (preferências)
     └─ Drawer: cabeçalho (avatar, username, LVL, barra de XP, 6 barras de skill, energia)
          + itens: Recent Activity, My Profile, Goals, Notifications, Friends, High Scores,
            How To Play, Donate/Remove Ads   (offline: sem Notifications, Friends, High Scores)
```

Detalhes por tela:

- **Feed de atividade**: cards com avatar, username, LVL, data relativa ("3 hours ago"), ícone da categoria, nome do achievement, foto, descrição, XP e skills ganhos, contagem de comentários. Tipos: `task`, `levelup`, `userreg`. Paginação por `lastx`. Filtros por achievement (`view_tasknum`) e por usuário (`profiler`).
- **Perfil**: avatar, LVL, barras, "XP earned", "Average XP per day", "Achievements completed", "Avg. completed per day", "Unique achievements", "Leveling up since", "High Scores rank". Perfis privados visíveis só a amigos.
- **Friends**: abas Friends | Search (busca por username ou e-mail). Estados: `notfriends`, `waiting`, `needapproval`, `friends`, `self`.
- **Notifications**: tipos `friendrequest` (botões ACCEPT/IGNORE), `newfriend`, `comment`, `reply`. Contador de novas via `lastchecked`.
- **High Scores**: abas Friends | Community; períodos PAST WEEK, PAST MONTH, ALL TIME; mostra `myrank`.
- **How To Play**: legenda de ícones + texto de ajuda longo (reproduzido em `strings.xml` do APK).
- **Donate/Remove Ads**: 5 opções de doação (produtos `donation_1..5`); remove anúncios para sempre no app e site, sem outros perks.
- **Settings**: foto de perfil; auto-hide; fuso horário (lista fixa de 35); alerta de notificações no app; e-mail (newsletter, alterar e-mail); privacidade: ocultar atividade do feed da comunidade, não participar do ranking, perfil privado, bloquear comentários.

## 5. Modelo de dados (banco local greenDAO, espelho do servidor)

| Tabela | Colunas |
|---|---|
| TASK_LIST (catálogo) | PK, TASKNUM, CAT, SUBCAT, TASK, POINTS, LVL, SKI1..SKI6 (STR,CUL,ENV,CHA,TAL,INT), LIMIT, WHO (0 = oficial, id do usuário = custom), TIME, TIME_EDITED, TIME_DELETED |
| USERTASKS (conclusões do usuário) | PK, ID, WHO, TASKNUM, WHEN, WHEN_EDITED, WHEN_DELETED, DESCRIP, GAINEDXP, SKI1..SKI6, SYNCME_IMAGE, SYNCME_CREATED, SYNCME_EDITED, SYNCME_DELETED |
| USERTASKS_NOT_OWN (cache do feed) | ID, VIEWTYPE, WHO, WHO_FBID, ACTIVITY_TYPE, TASKNUM, TASK_NAME, TASK_CATCODE, WHEN, DESCRIP, GAINEDXP, SKI1..SKI6, COMMENTCOUNT, COMMENTS_BLOCKED, USERS_LEVEL, USERNAME |
| HIGHSCORES | IS_FRIENDS, TYPE (semana/mês/tudo), USERID, FBID, USERNAME, RANK, XP, LVL |
| GOALS | TASKNUM, WHO |
| HIDDEN_TASKS | TASKNUM, WHO |
| COMMENT_COUNT | ID (usertask), COUNT |

Observações: soft delete em tudo (`when_deleted`); flags `SYNCME_*` permitem uso offline com sincronização
posterior; estatísticas (XP, nível, skills, energia) são recalculadas no cliente a partir de USERTASKS.

Preferências locais relevantes: LOGGEDINID, LOGINKEY, USERNAME, FBID, JOINED, pref_timezone (`"+1 hours#Europe/Amsterdam"`),
pref_autohide, pref_hideactivity, pref_hidehiscores, pref_hideprofile, pref_nocomment, pref_emailnews, pref_emailset,
pref_donated, PLAYOFFLINE, PREF_TIMESHITMAXTASKS, notifications_lastchecked.

## 6. API do backend original

Autenticação: `loggedinid` + `loginkey` (chave de sessão devolvida no login) enviados em cada requisição de escrita.
Senha de cadastro enviada como MD5 (`signup_pass_encrypted`); login envia a senha em claro via HTTPS. Datas em
`yyyy-MM-dd HH:mm:ss` UTC. Respostas JSON embrulhadas em um array com o nome do recurso, ex.: `{"login":[{...}]}`.

Hosts: `A` = `api1.lvluplife.com/app/v5/`, `P5` = `lvluplife.com/app/v5/`, `P3` = `lvluplife.com/app/v3/`.
Avatares: `lvluplife.com/userpic/thumb/{userid}/{fbid}` e `/userpic/large/...`.

| Endpoint | Método | Parâmetros | Resposta (campos lidos pelo app) |
|---|---|---|---|
| A `a_login.php` | POST | username, password, [users_fbid] | login[]: loggedinid, username, email, loginkey, joined_datetime, timezone, hideactivity, hidehiscores, hideprofile, autohide, emailnews, nocomment, donated, friendcount; erro: message, errorcode (`fbnotregistered`) |
| P5 `b_signup.php` | POST | signup_type=REG\|FB, signup_username, signup_email, signup_pass_encrypted(md5) ou fb_username, fb_email, fb_num | signup[]: success, errorcode |
| P5 `b_passreset.php` | GET | logininfo | passreset[]: success, msg |
| A `a_activity_mainuser.php` | GET | whoview=me, loggedinid, loginkey ou profile_hidden | activity[] (usertasks do usuário: id, who, tasknum, when, when_edited, when_deleted, descrip, gainedxp, str_gain..int_gain), customs[] (tasknum, cat, subcat, task, points, lvl, STR..INT, limit, time, time_edited, time_deleted), comments[] (commenton, count), goals[] (tasknum), hiddentasks[] (tasknum) |
| P5 `a_activity_2018.php` | GET | whoview=me\|friends\|community, loggedinid, [lastx], [achievement], [view_tasknum], [profiler] | activity[]: activity_type, usertask_id, who_id, who_username, who_fbid, who_lvl, tasknum, task, cat, description, when, xp, str..int, usertaskpic, commentcount, nocomment, max_completed_text; profileinfo[]: pid, friendcount, p_hidden |
| P5 `a_profileinfo_2018.php` | GET | loggedinid, profiler | profileinfo[]: pid, friend_status, p_hidden, curlvl, fbid, xp_perc, xp_min, xp_max, str..int, *_perc, xpEarned, avgxpperday, sinceDateTime, daysJoined, taskcount, p_unique_count, p_hidehiscores, rank99 |
| P5 `a_comments_2018.php` | GET | loggedinid, taskid | usertaskID, whostask_id, whostask_username, whostask_nocomment, comments[]: comment_id, comment, datetime, difference_txt, who_commented_id/_fbid/_username/_level |
| P3 `b_addcomment.php` | POST | usertaskid, whostask, comment | success |
| P3 `b_deletecomment.php` | POST | commentid | success |
| P5 `a_main_getnotifications_2018.php` | GET | loggedinid, lastchecked | ncount, notifications[] |
| P5 `a_notifications_2018.php` | GET | loggedinid | itemsFound, notifications[]: type (friendrequest\|newfriend\|comment\|reply), datetime, p_userid, p_username, p_fbid, p_lvl, comment, commenton, comment_whostask, comment_whostask_username |
| A `a_friends.php` | GET | loggedinid, [friendsearch] | friends[]: userid, username, fbid, status, level (ou friend_id, friend_username, friend_fbid, friend_status, friend_lvl); itemsFound |
| P3 `b_friendaction.php` | POST | dowhat=addfriend\|acceptfriend\|removefriend, friend | jsonaction[]: success, errorcode |
| P5 `a_highscores_2018.php` | GET | loggedinid, [view=friends] | highscores[]: myrank, week[], month[], all[] (userid, fbid, rank, username, lvl, xp) |
| P5 `b_taskeditornew.php` | POST | editornew=new\|edit, tasknum ou taskid, newdesc, datetime, when_edited, gainedxp, STR..INT, rotatedegrees, vnum=67, [image base64] | taskedit[]: success, newtaskid, imageuploadsuccess |
| P5 `b_deletetask.php` | POST | usertaskid, when_deleted — ou type=deletecustom, customtasknum, when_deleted | success |
| P5 `b_deleteUTP.php` | POST | usertaskid, when_pic_deleted | success |
| P5 `b_newcustom.php` | POST | task, edittasknum, cat, subcat, lvl, set_xp, set_str..set_int, set_limit | newcustom[]: success, newtasknum |
| P3 `b_goalset.php` | POST | tasknum, remove=yes\|no | success |
| P3 `b_hideset.php` | POST | tasknum, hiding=yes\|no | success |
| P3 `b_profilepic.php` | POST | image base64 | profilepic[]: success, failcode, info |
| P3 `b_deleteprofilepic.php` | POST | — | profilepic[]: success |
| P5 `b_settingchange.php` | POST | key (pref_*), val | success |
| P5 `b_reportuser.php` | POST | whobad, reason 1..5, comments | success |

Status em 2026: `api1` fora do ar; em `lvluplife.com` só `a_login.php` responde (com warnings de PHP); app
removido da Play Store em 13/12/2023.

## 7. Monetização e integrações

- Banners AdMob em: categorias, lista de tarefas, feed, formulário de conclusão, notificações, ranking. Interstitial na lista de tarefas após bater o limite horário repetidamente.
- Doações (in-app products `donation_1..5`) marcam `donated` no servidor e removem anúncios no app e no site. Nenhuma feature é paga.
- Login com Facebook (app id 244330595731426) e avatar via fbid.
- Firebase Analytics registra `select_content` para cada chamada de API.

## 8. Assets reaproveitáveis como referência de design

No APK (`res/drawable`): ícones pixel art de categoria (`cat_art, cat_car, cat_fit, cat_foo, cat_fun, cat_goa, cat_hou,
cat_hum, cat_men, cat_mus, cat_out, cat_pho, cat_rea, cat_sch, cat_soc, cat_spo, cat_top, cat_tra, cat_wri, cat_locked`),
ícones de ação (`ic_quill, ic_bomb, ic_scroll, ic_lock, ic_key, ic_star, ic_stargone, ic_help`), balões de comentário
`cmt0..cmt10`, `heart.png`/`heartgone.png`, `levelup.png`, `logo.png`/`logobig.png`, `alertcube.png`, `dialog_icon.png`,
`fab_plus_icon.png`, fundos `bubble`, `button_*`, `editbox`, `bar_xp`, `bar_skills`. Esses arquivos são de autoria
do desenvolvedor original; servem de referência, não devem ser copiados para o produto final.

## 9. Problemas e dívidas do original a não repetir

- Senha em MD5 sem salt no cadastro e senha em claro no login; `loginkey` sem expiração conhecida.
- Autorização fraca: várias chamadas de leitura só exigem `loggedinid` (perfil, atividade, comentários).
- Catálogo duplicado no código (567 entradas hard-coded) e no servidor, com `vnum` fixo.
- Nível máximo 60, mas conteúdo acaba no nível 20; reclamação recorrente dos usuários (categorias esgotam ~nível 13).
- Datas como strings UTC manipuladas à mão; fuso horário configurado por lista fixa.
- Estatísticas recalculadas no cliente a partir do histórico completo, sem agregação no servidor.
- Dependência de Facebook SDK, Fabric (descontinuado) e Support Library (substituída por AndroidX).

## 10. Lacunas que ainda dependem de prints ou do servidor

- Aparência final das telas (cores aplicadas, espaçamentos, animações): prints do app ajudam.
- Fórmulas de `rank99`, `avgxpperday` e `max_completed_text` são calculadas no servidor; não estão no APK.
- Conteúdo exato das notificações push (FCM registrado, mas sem handler de mensagens no app).
