# AUDIT DSP — qosmo-dsp, 2026-09-03 (apres-midi)

> Suite de `AUDIT_DSP_20260903.md` (le matin). Meme perimetre, meme methode :
> lecture du code, recoupee avec les mesures de l'apres-midi (budget 8000/16000/
> 32000, mailbox.log, DERAIL-ZERO, SP-RING). Ce qui est deja etabli le matin n'est
> pas repete, sauf pour le corriger. Toute affirmation sur un mecanisme cite
> `fichier:ligne` verifie par `grep -n` sur l'arbre du 03/09 17:45. Ce que je n'ai
> pas verifie est au §7.

---

## 1. Chiffres a l'ouverture

| grandeur | matin | apres-midi | source |
|---|---:|---:|---|
| code Calypso (`hw/arm/calypso/*.c *.h`) | 40 011 | **33 978** | `wc -l` |
| `calypso_c54x.c` | 21 296 | 21 274 | idem |
| `calypso_trx.c` / `_bsp.c` / `_rhea_dma.c` / `_rif.c` / `_mailbox.c` | — | 2 496 / 2 391 / 689 / 481 / 377 | idem |
| `hw/char/calypso_uart.c` | — | 925 | idem |
| `tools/qosmo-launch/qosmo-launch.c` (nouveau) | — | 920 | idem |
| sites de gate dans `c54x.c` : `getenv` / `calypso_gate` / `cdbg_env` / `calypso_debug_enabled` | — | 120 / 142 / 16 / 78 = **356** | `grep -c` |
| `fprintf(stderr` dans `c54x.c` | — | **672** | idem |
| `@BEQUILLE` | 119 (72 hors shunt) | **61** (42 dans `c54x.c`) | `grep -c` |
| variables `CALYPSO_*` distinctes declarees dans `environnement/*.env` | 279 (code) | **354** (dont 171 en ligne active `: "${X:=...}"`) | `grep -oE` |
| variables `CALYPSO_*` lues par le modele (`hw/arm/calypso` + `uart.c`) | — | **324** | idiomes `getenv/calypso_gate/cdbg_env/parse_uint_env` |
| declarees mais lues par personne (modele, scripts, outils) | — | **60 actives** (+ 64 en commentaire) | `comm` (§4) |
| lues par le modele mais declarees dans aucun `.env` | — | 94 | idem |

Le comptage des lectures ne voit que les idiomes listes ; `CALYPSO_CORRELATOR_TRACE`
est lue par un idiome indirect (`c54x.c:795`) et sort a tort en « morte » — marge
d'erreur de quelques unites, pas de dizaines.

---

## 2. Ce qui a change depuis le matin (V1..V4, etat verifie)

| verrou | matin | etat 17:45 | preuve |
|---|---|---|---|
| **V1** IT trame sur vec 19/3 | corrige | **confirme applique** : emission sur 28/12 a la source | `calypso_trx.c:1864` `c54x_interrupt_ex(s->dsp, C54X_IT_TPU_FRAME_VEC, C54X_IT_TPU_FRAME_BIT)` ; `calypso_c54x.h:200-201` (28/12) ; plus aucun `FRAME_VEC28`/`FRAME_IT_NATIVE` dans le code (restent 2 lignes mortes en `.env`, §4) |
| **V2** SCH a sortie constante | ouvert, 3 hypotheses | **ouvert, reformule** : la sortie n'est plus « constante » (0xf8d8 OU 0x0000) et un SB a ete decode une fois (BSIC=21). L'hypothese « opcode du Viterbi » recule, l'hypothese « mauvais burst dans le tampon » avance (§3a, §3d) | mesure du jour + `0xb214` = `rpt ; mvdd` (copie de bloc, pas un decodeur) |
| **V3** vecteur de livraison RX (21/19 vs 16/30) | a departager | **non teste aujourd'hui** ; le chemin par defaut emet toujours vec21/vec19 | `calypso_bsp.c:1146-1169` (`calypso_bsp_deliver`, reroute seulement si `CALYPSO_BSP_RX_VEC`) |
| **V4** `a_pm` substitue | tranche | en place : `calypso_bsp_rssi_apm()` | `calypso_bsp.c:154` |
| §4 chemins portes hors shunt | fait | `c54x_task_md()` `c54x.c:20526` ; `c54x_early_boot()` `c54x.c:20491` appele `calypso_mb.c:256` | grep |

Correction au matin : « ~500 k insn/s » (README:47, audit §2) — mesure du jour
**0,43 M insn/s**, invariant au budget.

---

## 3. Les verrous, par ordre de rendement

### 3a — Le modele temporel tick/trame : le DSP recoit N trames radio par tick et n'en traite qu'une

**Symptome mesure.** `dsp_n_exec_5 == budget` sur toutes les lignes `[tdma]` a 16000
et 32000 ; 8 trames GSM par tick a 16000, 16 a 32000 ; overrun RIF 75 % a 16000/32000,
1,8 % a 8000 ; 31 % des `d_fb_det=1` effaces par `0xb2cc` avant lecture ARM.

**Mecanisme, dans le code.**

| maillon | ou | ce que ca fait |
|---|---|---|
| horloge du tick | `calypso_trx.c:1692-1709` | `CALYPSO_TDMA_REALTIME=1` → `QEMU_CLOCK_REALTIME`. **Le depot le pose a 1** (`environnement/calypso.env:59`), alors que le commentaire du code (`trx.c:1695-1702`, « DEFAULT VIRTUAL, single-domain ») et `ETAT_ACTUEL §12.5` (« CORRIGE par TDMA_REALTIME=0 ») decrivent l'autre mode. Le mode qui tourne est REALTIME. |
| maitre du FN | `trx.c:1763-1770` (REALTIME) | `s->fn = g_wall_fn` du pthread `clk_master_loop` (`trx.c:1445`), qui avance a 4,615 ms **mur** quoi qu'il arrive, et envoie le CLK a la radio a ce rythme |
| budget par tick | `trx.c:1807-1813` | defaut code 256000, **le depot pose 32000** (`bsp.env:33`) ; deux appels `c54x_run(s->dsp, dsp_budget)` (`trx.c:1821` boot, `trx.c:1877` RX) |
| duree d'un tick | 0,43 M insn/s | 8000 → 18,6 ms ; 16000 → 37 ms ; 32000 → 74 ms — soit **4 / 8 / 16 trames GSM par tick**. C'est exactement la mesure « 8 trames a 16000, 16 a 32000 » |
| rattrapage | `trx.c:2058-2062` | `while (target <= now) { target += GSM_TDMA_NS; skipped++; }` : les trames en retard sont **sautees**, le FN radio a avance de N |
| ce que voit l'ARM | `trx.c:2011` | **une** IRQ trame par tick (`qemu_irq_raise(CALYPSO_IRQ_TPU_FRAME)`), et `calypso_timer_lost_frame_tick(s->fn)` (`trx.c:2010`) fige le timer pour que `check_lost_frame()` ne crie pas. L'ARM compte donc **1 trame par tick** pendant que la radio en compte N |
| ce que voit le DSP | `trx.c:1847-1867` puis `1876-1878` | une IT trame (bit 12 de l'IFR : N ITs arrivees pendant un tick long **se replient en une**), puis `budget` instructions. `dsp_n_exec_5 == budget` = le DSP n'atteint jamais IDLE dans le budget (`c54x.c:14931` boucle `!s->idle`) : son travail par trame depasse 32000 insn |
| le yield | `c54x.c:20399` (defaut 32768), `20423-20425` | ne coupe qu'a `executed >= 32768` (INTM=0) ou `4x` : **inerte sous 32768**. Avec 32000 c'est le budget qui plafonne, pas le yield. `ETAT_ACTUEL §12.5` (« DSP_BUDGET MORT, le vrai plafond est YIELD ») etait vrai au defaut 256000, **faux depuis `bsp.env:33`**. Le commentaire `trx.c:1801` (« 256000 ≈ 1 trame nominale ») est lui aussi contredit : 256000 insn = 0,6 s mur |
| arrivee des bursts | `calypso_bsp.c:1369` (timer REALTIME 5 ms), `977-996` (`bsp_drain_cb`, jusqu'a 64 bursts par passage), `937-957` (`BSP_DIRECT_FEED=1`, « SANS match FN », `calypso.env:280`) | les timers QEMU tournent sur le **meme thread** que `c54x_run` : pendant un tick de 37 ms rien n'est draine, puis 8 bursts sont pousses **a la suite** dans `calypso_rif_rx_burst` (`c54x.c:21257`), chacun ecrasant l'etage (`rif.c:400-406`, overrun compte). Le DSP ne voit que le dernier |
| DMA de reception | `rif.c:449-465` → `rhea_dma.c:386` | le drain vers `0x0cce` est **synchrone dans le push** ; il n'a lieu que si le DSP a re-arme (`ENABLE`/`DIRECTION`, `rhea_dma.c:424-437`). Un DSP qui ne repasse pas entre deux bursts ne re-arme pas → overrun. D'ou 75 % a 16000/32000 (8-16 bursts d'affilee) et 1,8 % a 8000 si, a ce budget, le DSP atteint IDLE et le tick tient ses 4,6 ms (**a verifier**, exp. 1) |

**Pourquoi 31 % des detections FB sont effacees.** Ordre dans un tick : (4) IT trame
DSP → (5) `c54x_run` ou la ROM execute le prologue de tache `0xb2cc` (`st *(0x08f8),#0`
puis `0x08fa..0x08fc` a zero — verifie au desassembleur) puis, si detection, `0x79e4`
`orm *(0x08f8),#1` → (7) IRQ trame ARM (`trx.c:2011`) → retour. L'ARM ne lit qu'une
fois le tick rendu ; si le tick suivant est **deja du** (work_dt > 4,6 ms, cas des trois
budgets), il repart aussitot et son (5) efface `d_fb_det` avant que l'ARM ait tourne.
La fenetre ARM n'existe que quand le vCPU obtient du temps entre deux ticks.

**Pourquoi la tache SB traite le mauvais burst.** L'ARM programme SB pour *sa* trame
(1 par tick) ; le burst present dans l'etage RIF est le dernier arrive en temps mur
(FN radio, N par tick, aucune correspondance : `bsp.c:934-935`). La position du SCH
dans le multitrame 51 est donc aleatoire vue de la tache SB : CRC faux presque
toujours (`a_sch[0]=0x8100` = `B_BLUD|B_SCH_CRC`, bits 15/8 de `l1_environment.h:269-270`),
juste une fois (BSIC=21, parasite). Le gate `CALYPSO_RIF_FCCH_ONLY` (`c54x.c:21233-21257`,
defaut 0) filtre precisement sur `calypso_trx_get_fn() % 51` — c'est l'experience
toute prete (§6).

**Hypothese testable.** Le DSP est en retard parce qu'il recoit `budget` insn par
tick pour un travail par trame superieur, et le tick est plus long qu'une trame.
Deux leviers, mesurables sans recompiler : `CALYPSO_TDMA_REALTIME=0` (le tick
devient maitre du FN, `trx.c:1746-1761` : 1 trame par tick, radio ralentie mais
alignee) ; budget tel que le DSP atteigne IDLE (`dsp_n_exec_5 < budget`).

**Instruments presents.** `[tdma]` (`trx.c:1914-1922`, 1/1000 ticks), `[rif] burst #`
(`rif.c:467-475`, overrun), `[rearm-fix]`/`[tdma-skip]` (`trx.c:2069-2081`, `skipped`),
`CALYPSO_TIMER=1` → `/tmp/tdma_tick.log` (`trx.c:2094`), mailbox (`0x08f8` DSP>WR vs ARM<RD).

### 3b — DERAIL-ZERO : un RET/CALA depile ou lit 0x0000

**Symptome mesure.** `DERAIL-ZERO` depuis `0xdb70` (16000, fatal : plus d'IDLE, plus
d'IRQ API, FBSB timeout), `0x8ed2` x3 (32000, non fatal), `0xd963 op 0xf5e3` (gta0x),
0 en 5 min a 8000. Retour par le reset-handler `0xb410` puis boucle bootloader
`0xb41c` (poll `data[0x0fff]`, INTM=1).

**Ou est le RET/CALA fautif.** La sonde `c54x.c:17688-17698` logge `from=g_prev_pc`,
`op=g_prev_op`, les 4 mots ROM precedents, et au 1er evenement `SP-RING-AT-STORM`
(`17703-17722`) : les 64 dernieres operations ayant touche SP, avec pour chacune
`delta` mesure vs `exp` attendu (`RET +1`, `CALA -1`, `FRET +2`...) et un marqueur
`MISMATCH`. Le ring est alimente **sans gate** a chaque changement de SP
(`c54x.c:18125-18134`). `0xf5e3` = `cala B` : a `0xd963` comme a `0xaffc`
(`ld *(0x3f64),B ; cala B`), la cible vient d'une **cellule de donnees**, pas de la
pile — un slot a zero saute en 0x0000 sans que SP soit fautif. Les deux mecanismes
(pile / pointeur nul) se distinguent dans la trace par `op` : `0xfc00` (RET) contre
`0xf5e3` (CALA).

**Trois pistes, dans le code.**

| piste | ou | ce qu'on lit |
|---|---|---|
| **1. desequilibre de pile sur une IT** | entree : `c54x.c:15550-15563` (rejeu in-loop), `21001-21013` (reveil IDLE), `21044` (normal) — toutes poussent **2 mots** (PC, XPC). Sortie `RETE` : `c54x.c:8062-8073` — depile XPC **seulement si le sommet <= 3** (heuristique), 2 mots inconditionnels seulement sous `CALYPSO_RETE_POP2=1` (defaut 0, `8062`) | push inconditionnel contre pop devine : le commentaire `8035-8058` le dit lui-meme, avec la mesure `-4 mots/trame`. Un PC de retour <= 3 est rare, mais un XPC valant 0 empile puis un PC... le cas inverse (sommet = PC bas ou garbage) casse la parite. **Piste n.1** : elle explique pourquoi le taux depend du budget (plus d'ITs repliees/rejouees par tick) |
| **2. IT prise au milieu d'un RPT ou d'un delay-slot** | le rejeu in-loop `c54x.c:15550-15553` ne teste **que** `INTM` : ni `delay_slots` ni `rpt_active` (aucune occurrence de `rpt_active` entre `14931` et `15560`). `rpt_active` n'est efface qu'a `20181` (compte epuise) et `20806` (reset). Les deux autres entrees testent `delay_slots` (`21044`, `6024`) mais pas `rpt_active` | une IT vectorisee pendant un `RPT` laisse `rpt_active` arme : **la premiere instruction de l'ISR est repetee** (un `PSHM` repete = pile decalee, un `POPM` repete = over-pop). Sur silicium `RPT` est non interruptible (le yield le sait : `20423`). Testable par comptage |
| **3. opcode mal decode / FRAME-POPM** | `RAPPORT_OPCODES.md` (plages `0x60-0x8F`, `0xC0-0xFF` sans passe de refutation) ; `POPD` manquait encore le 23/07 (`c54x.c:12652-12658`, « RET->0 -> DERAIL-ZERO from=0x0157 ») | precedent direct : un pop non implemente a deja produit ce symptome. Le ring SP-RING marque `NON-XFER touche SP` pour tout op hors transfert : c'est la ligne a lire en premier |

**Instruments presents.** `DERAIL-ZERO` + `SP-RING-AT-STORM` (inconditionnels, cap 60),
`CALYPSO_DEBUG=BOOTSTUB` → `BL-REENTRY` (`c54x.c:7343-7370`, entree anormale en `0xb41c`
avec `prev_pc`, pile, trail), `CALYPSO_ORPHAN=1` (`2666`, `18158` : retour qui lit une
valeur non-retour), `CALYPSO_RETE_AUDIT=1` (`8080`), `CALYPSO_SP_RING=1` (`14153`),
`POST-BOOTSTUB-RET` (`10512-10522`, inconditionnel : c'est lui qui fait 2,8 Mo/s quand
ca deraille, `40-qemu.sh:10-14`).

### 3c — Selection de page MCU→DSP : qui a raison ?

**Mesure.** L'ARM ecrit surtout la page 1 (`0x081f d_ctrl_abb`, `0x0823 d_afc` : 9 350
ecritures) ; le DSP lit surtout la page 0 (`0x080b/0x080f` : 1 354) et la page 1 335 fois
(`0xb5a5/0xb5ad`) ; `d_dsp_page` : 0→2, 2→3, 2→0.

**Le protocole (osmocom-bb).** Adresses : W page 0 `0xFFD00000` → mot DSP `0x0800`,
W page 1 `0xFFD00028` → `0x0814`, R page 0/1 → `0x0828`/`0x083c`, NDB `0xFFD001A8` →
`0x08d4` (`dsp_api.h:18-23`). Donc `0x081f = 0x0814+11 = d_ctrl_abb page 1`,
`0x0823 = d_afc page 1` ; `0x080b/0x080f` = les memes en page 0. `B_GSM_TASK = bit 1`
(`l1_environment.h:249`) ; `dsp_end_scenario()` ecrit `d_dsp_page = B_GSM_TASK | w_page`
**puis** `w_page ^= 1` (`calypso/dsp.c:471-472`) ; `w_page` ne bascule **que** la (appelee
si `TDMA_IFLG_DSP`, `sync.c:275-276`). Chaque trame l'ARM efface `db_w` et y pose
`d_afc` (`sync.c:244-247`).

**Lecture.** Bit 0 de `d_dsp_page` = la page que le DSP doit lire (derniere scenario
close) ; l'autre page est celle que l'ARM remplit. Quand aucun item DSP n'est planifie
plusieurs trames de suite, `w_page` ne bascule pas : l'ARM reecrit `d_afc` dans la
**meme** page (ici la 1, 9 350 fois) sans le signaler, et le DSP continue de lire la
page nommee par le dernier `d_dsp_page` (la 0). **Les deux ont raison** ; l'asymetrie
9 350/1 354 dit seulement que l'ARM planifie peu d'items DSP — coherent avec un ARM
qui attend des reponses FB qui n'arrivent pas dans sa fenetre (3a). `2→3` puis `2→0`
(reset L1, `sync.c:165/309`) sont les seules transitions attendues.

**Ce que le modele fait en plus (a verifier).** `api_ram == dsp_ram`
(`trx.c:2394`, `c54x.c:16533`) : l'ecriture ARM est directement visible du DSP, et le
« miroir » `trx.c:1389` est une auto-affectation sans effet. Mais `calypso_dsp_done()`
(`trx.c:1349-1389`, declenche par `TPU_CTRL.EN`, `trx.c:1532-1538`) choisit la page par
`dsp_ram[0x01A8/2] & 1` (`1365`) et **copie la page W dans la DARAM `0x0584/0x0586..`**
(`1384-1387`). Or la ROM fait elle-meme une copie page W → tampon (`0xb001-0xb00b`,
six paires `ld *AR1(k),A / stl *AR2(k),A`). Deux copistes pour un tampon : si le
DSP acquitte `d_dsp_page` (efface bit 1) et que le modele le lui reimpose, la tache
est rejouee a chaque trame — ce qui collerait avec 404 `d_fb_det=1` pour 280 lectures
ARM et avec la file DMA qui deborde (3e). **Test** : `grep 'DSP>WR 0x08d4' mailbox.log`
(le DSP ecrit-il la cellule ?) et `CALYPSO_WMAP=1 WMAP_LO=0x0584 WMAP_HI=0x059a`
(qui ecrit le tampon : PC ROM ou hote).

### 3d — Le decodeur SCH

**Mesure.** `a_sch[3]` (`0x084e`, page R 1) ecrit a `PC=0xb214` : `0xf8d8` ou `0x0000` ;
`a_sch[0]=0x8100` ; un SB decode une fois (BSIC=21).

**Ce que dit la ROM.** `0xb213 rpt ; 0xb214 mvdd` : `0xb214` est une **copie de bloc**
(memoire vers memoire) vers la page R, pas le decodeur. Le producteur de `0xf8d8` est en
amont, dans le tampon source du `mvdd` (AR non identifie ici). `a_sch[0]=0x8100` :
l'ARM pose lui-meme `1<<B_SCH_CRC` (`0x0100`) a chaque bascule de page R (`sync.c:269`) ;
le DSP y ajoute `B_BLUD` (bit 15) — il a donc **execute** la tache et rapporte
« bloc present, CRC faux ». Un CRC faux sur un burst qui n'est pas le SCH (3a) est le
comportement attendu ; le BSIC=21 unique montre que la chaine sait produire un
mot different quand l'entree est la bonne. La sonde `SBSLOT-WR` (`c54x.c:5099-5118`,
cap 300) capture `A` et `op` : elle permet de remonter d'un cran (source du `mvdd`).

**Hypothese.** Entree, pas decodeur. A departager par `CALYPSO_RIF_FCCH_ONLY=1` (§6) :
si la proportion de CRC OK monte, V2/2 (opcodes) est ecartee pour le SB.

### 3e — `DSP_ERR_DMA_PROG` (= 8) permanent

**Mecanisme (documente dans le code).** `rhea_dma.c:400-416` : la ROM (`0xaa83/0xaaad/
0xaad1`) tient trois files circulaires de 14 entrees et leve `DMA_PROG/TASK/PEND` quand
elles **debordent** ; l'ARM imprime et efface (`sync.c:249-251`), le DSP repose. Le
transfert existe (`CALYPSO_RHEA_DMA_XFER=1`, `modes.env:100` ; `rhea_dma.c:386-600`,
296 mots en 4 pages contigues `549-560`, `IRQ_STATE` `588`, INT10n en niveau
`rhea_dma.c:376-384` → `c54x.c:5924`). Ce qui manque est la **cloture cote ROM** :
`c54x.c:4068-4080` (mesure du 04/08) — INT10n levee 15 500 fois, vecteur 30 entre,
mais le tremplin `data[0x0158]` ne s'execute que 8 fois et son corps `0x728a` jamais :
**personne ne depile la file**. Le temoin ne dit donc pas « DMA casse » mais « ISR de
fin de DMA jamais atteinte ». Sous 3a, une seule IT trame et un seul run par tick
laissent peu de fenetres INTM=0 pour une IT de niveau.

**Instrument.** `TRAMPO` (`c54x.c:4095`, ecritures sur `0x0158-0x015f`),
`CALYPSO_MAILBOX_CELLS=0x3f92` (le mot d'etat interne dont `d_error_status` est la
copie : `DSP_ARM_LINKAGE.md`), `CALYPSO_DEBUG=C54X` pour `vec=30`.

---

## 4. Dette de configuration, code mort, incoherences doc/code

**Variables `.env` actives sans aucun lecteur (60)** — familles, fichier porteur :
`CALYPSO_SHUNT_*` (10, `calypso.env`/`modes.env`), `CALYPSO_INJECT_{AGCH,SACCH,SDCCH,TCH}`,
`CALYPSO_RECORD_FULL*` (5), `CALYPSO_TCH_*` (5), `CALYPSO_ARM2DSP_{BGEN_*,CTRLSYS_*,TASKBIT,TASKWORD}`
(11, `armdsp.env`), `CALYPSO_TWL_ACK_{ARM,DSP}`, `CALYPSO_FRAME_IT_NATIVE`, `CALYPSO_BSP_DARAM_FORCE`,
`CALYPSO_CANNED`, `CALYPSO_NATIVE`, `CALYPSO_FORCE_{0810,3F92,DEMOD_BRIDGE}`, `CALYPSO_FB_IQ_{BASE,DARAM}`,
`CALYPSO_FBSB_TIMEOUT`, `CALYPSO_START_FN`, `CALYPSO_SCAN_TO`, `CALYPSO_LAST_RACH_FN_ADDR`
(exportee par `16-fwsyms.sh:88` et le lanceur, lue nulle part depuis le retrait du shunt).
Repartition : `calypso.env` 31, `modes.env` 15, `armdsp.env` 12. Liste complete :
`comm -23` des deux inventaires (§1).

**Sondes qui coutent sans gate, sur le chemin chaud de `c54x.c`.**

| sonde | ou | gate | cout (lu dans le code) |
|---|---|---|---|
| `INTM-TRANS` | `17982-17991` | `CALYPSO_INTM_TRANS` — **:=1 dans `calypso.env:347`** (aussi `calypso_wire.env:33`) | un `fprintf` par transition INTM ; `paths.env:112` chiffre « 56 % du flux » de qemu.log. Contredit `debug.env:5` (« toutes desactivees par defaut ») |
| `CORRELATOR_TRACE` | `1787-1794`, `16806` | `CALYPSO_CORRELATOR_TRACE` — **:=1 dans `calypso.env:346`** | une comparaison par lecture memoire + insertion table de hachage quand PC ∈ [0x8d00,0x9000) |
| `FBDET-WR` | `3616-3619` | **aucune** | `fprintf` a chaque ecriture de `0x08f8` : le prologue `0xb2cc` en fait une par trame |
| `POST-BOOTSTUB-RET` | `10512-10522` | **aucune** (200 puis 1/50) | 2,8 Mo/s mesure en deraillement (`40-qemu.sh:10-14`) |
| `LEVELCHK-EMPIRICAL` | `5948-5978` | aucune, cap 5000 lignes | calcul par instruction, `fprintf` borne |
| `SP-event ring` + `sp_ledger` | `18125-18134` | aucune | 4 stores par changement de SP, pas de sortie : c'est le prix de `SP-RING-AT-STORM`, a garder |
| histogrammes `g_ar_hist`, `g_sp_abs_hist`, `g_stuck_hist`, `g_corr_read_hist` | `233`, `624`, `926`, `777` | gates respectifs (`SP-ABS` `633`, `STUCK-PROBE` `941`, ...) | listes lineaires (`for` sur `*_MAX`) : couteux **si** actifs ; eteints par defaut sauf corr (ci-dessus) |
| `XPC-STATS` | `14484-14520` | `calypso_debug_enabled` pour la sortie, comptage inconditionnel | une fois par `c54x_run`, negligeable |
| `FEED-DST` | `1672-1730` | `CALYPSO_FEED_DST` — **:=1 dans `bsp.env:231`** | compteurs par lecture dans deux plages, plafonne |

Le debit 0,43 M insn/s n'est pas decomposable depuis le code ; seul un A/B
(`CALYPSO_INTM_TRANS=0 CALYPSO_CORRELATOR_TRACE=0`) le mesure (§6, exp. 3).
Ordre de grandeur : un C54x a 65 MHz (`c54x.c:13978`) ou 104 MHz (`trx.c:1802`, les deux
commentaires se contredisent) fait 300-480 k cycles par trame ; a 0,43 M insn/s une
trame de DSP reel vaut **0,7-1,1 s mur**.

**Mailbox.** `CALYPSO_MAILBOX:=1` (`debug.env:80`) ; ~8 000 lignes/s a 8000 ; plafond
`CALYPSO_MAILBOX_MAX` 5 000 000 (`mailbox.c:15`, `183`, coupe a `297-303` avec une ligne
`# PLAFOND`), atteint en ~8 min ; `fflush` toutes les 1024 lignes (`mailbox.c:367`).
Repli `xN` seulement sur valeur identique consecutive (`340-350`).

**Incoherences doc/code relevees.**

| ou | ce qui est ecrit | ce que le code / la mesure dit |
|---|---|---|
| `ETAT_ACTUEL §12.5` | `CALYPSO_DSP_BUDGET` « MORT », plafond = YIELD 32768 | vrai au defaut 256000 ; **faux depuis `bsp.env:33` (32000)** : `dsp_n_exec_5 == budget`. Le renvoi `c54x.c:16900` est perime (`20399`) |
| `ETAT_ACTUEL §12.5` | « CORRIGE par `TDMA_REALTIME=0` » | `calypso.env:59` pose **1** ; le mode qui tourne est REALTIME (a confirmer sur `qemu-manifest.log`, §7) |
| `trx.c:1695-1702, 1747-1751` | « DEFAULT VIRTUAL, le pthread n'est PAS demarre » | defaut **code** ; le depot l'inverse par l'env |
| `trx.c:1801` | « 256000 ≈ 1 trame nominale » | 256000 insn = 0,6 s mur |
| `trx.c:226` | « miroir par tick `api_ram[d_dsp_page] = dsp_ram[0x01A8/2]` » | `api_ram == dsp_ram` (`trx.c:2394`) : auto-affectation ; et elle est dans `calypso_dsp_done`, pas dans le tick |
| `debug.env:5` | sondes « toutes desactivees par defaut » | `INTM_TRANS`, `CORRELATOR_TRACE` (`calypso.env`), `FEED_DST` (`bsp.env`) sont a 1 |
| `dsp.env:24` | `DSP_BUDGET` « defaut run.sh :=256000 » | `bsp.env:33` :=32000 gagne (charge avant `calypso.env`, `load.env:37`) |
| `README:47` | « ~500 k instructions/s » | 0,43 M mesure |
| `c54x.c:8024-8033` (commentaire RETE) | « l'entree IT pousse XPC seulement si xpc!=0 » | les trois entrees poussent 2 mots inconditionnellement ; le commentaire suivant (`8035`) le dit deja — deux commentaires contradictoires cote a cote |
| `bsp.env:8-21` | `CALYPSO_BSP_DARAM_ADDR` = valeur « mesuree, ne pas changer » | la meme variable est redeclaree vide 30 lignes plus bas (`bsp.env:44`, inventaire) — sans effet, mais trompeur |

---

## 5. Ce qui a ete fait le 2026-09-03 apres-midi (verifie dans les fichiers, mtime > 15:20)

| ajout | fichier | verification |
|---|---|---|
| lanceur C `qosmo-dsp` | `tools/qosmo-launch/qosmo-launch.c` (920 l.), `Makefile` (`ALIAS/TREE/DSP`, `make install` → `/usr/local/bin`) | options `--bind/--trx-port/--iq-tee/--l1ctl/--monitor/--gdb` (`492-523`) ; symboles `l1s`/`last_rach` lus dans l'ELF (`642-653`, exportes `CALYPSO_L1S_FN_ADDR` lu par `trx.c:445`, `CALYPSO_LAST_RACH_FN_ADDR` lu par personne) ; liens `modem.pty`/`irda.pty` (`663-666`, `symlink` `547`) ; `CALYPSO_GDB_PORT` force (`669-670`) ; `--bind` → `CALYPSO_BSP_BIND_ADDR` (`672-676`) |
| `40-qemu.sh` passe par le lanceur | `run_modules/40-qemu.sh` (bloc « LANCEUR C », `--qemu ... -k ... --bin ... --gdb ... --rundir ... --monitor`) | repli sur la ligne historique si `/usr/local/bin/qosmo-dsp` absent ; `--bin "$FIRMWARE_BIN"` est transmis mais le stub romload ignore le contenu (ci-dessous) |
| `44-gdb-telnet.sh` + `tools/gdb-telnet.py` | nouveaux (17:41-17:43) | port 44444, `gdb-multiarch` attache au stub `CALYPSO_GDB_PORT` en `continue &`, une session a la fois ; gate `MOD_ENABLED_IF` sur `CALYPSO_GDB_PORT` |
| `43-mailbox-dissam.sh` | `setsid sh "$boucle"` avec commentaire `[2026-09-03]` (RUN_DIR sous `/run` noexec) | oui |
| `CALYPSO_MAILBOX` remis a 1 | `environnement/debug.env:80` + note `[2026-09-03]` | oui |
| fenetre tmux « asm » retiree, « dsp » a 3 panes | `tmux_modules/calypso.sh:36-40`, `_commun.sh:82-104` (3e pane : `mailbox-annote.py --flux`) | oui |
| pane SI conditionne au pipeline gr-gsm | `run_modules/80-panes.sh` (`calypso_pipeline_is_grgsm && BRIDGE != pont`) | oui |
| `start-direct.sh --dsp` → `CALYPSO_BRIDGE=none` | `/opt/GSM/osmo-operator/start-direct.sh:455-457` (hors depot) | oui, avec la note sur 5700-5702 |

**Stub romload** (`hw/char/calypso_uart.c:295-320`) : les blocs `<w` sont **consommes et
jetes** (`ROM_BLOCK_DATA`, `386-391` : simple decompte de `needed`), `>w` renvoye. Le
firmware qui tourne est celui de `-kernel`, ce qui rend le `.bin` d'osmocon inerte et le
firmware interchangeable (compal_e88/e86, gta0x testes) — coherent avec le code.

---

## 6. Trois prochaines experiences

**Exp. 1 — Un tick = une trame (3a).** Meme banc qu'aujourd'hui, budget 8000, mais
tick maitre du FN :
```
CALYPSO_TDMA_REALTIME=0 CALYPSO_DSP_BUDGET=8000 ./start-direct.sh --dsp --no-attach
grep -m1 'tdma_timer clock' /tmp/calypso/logs/qemu-manifest.log     # doit dire VIRTUAL
grep '\[tdma\] tick' /tmp/calypso/logs/qemu.log | tail -3            # dsp_n_exec_5 < budget ?
grep -c 'OVERRUN\|overrun=' /tmp/calypso/logs/qemu.log ; grep '\[rif\] burst #' /tmp/calypso/logs/qemu.log | tail -1
awk '$4=="0x08f8" && $3=="DSP>WR" && /-> 0x0001/' /tmp/calypso/logs/mailbox.log | wc -l   # d_fb_det poses
awk '$4=="0x08f8" && $3=="ARM<RD" && /= 0x0001/'  /tmp/calypso/logs/mailbox.log | wc -l   # ... lus a 1
```
Tranche : si le ratio lus/poses monte au-dessus de 69 % et `overrun` tombe, le retard
est bien le modele temporel, pas le DSP.

**Exp. 2 — Le deraillement est-il une IT prise en RPT ou une RETE dissymetrique (3b).**
Deux runs de 5 min a 16000 (le budget qui deraille) :
```
CALYPSO_DSP_BUDGET=16000 CALYPSO_RETE_POP2=1 CALYPSO_ORPHAN=1 CALYPSO_DEBUG=BOOTSTUB ./start-direct.sh --dsp --no-attach
grep -c 'DERAIL-ZERO' /tmp/calypso/logs/qemu.log
grep -A64 'SP-RING-AT-STORM' /tmp/calypso/logs/qemu.log | grep -c 'MISMATCH\|NON-XFER'
grep 'BL-REENTRY #1 ' /tmp/calypso/logs/qemu.log      # prev_pc + trail : la 1re entree anormale en 0xb41c
```
puis le meme run sans `RETE_POP2`. Tranche : `DERAIL-ZERO` a 0 avec `POP2` = piste 1 ;
sinon, dans `SP-RING-AT-STORM`, une ligne `NON-XFER touche SP` dont l'`op` est un
`PSHM/POPM` juste apres un vecteur (`pc` dans `0x0080-0x00ff`) = piste 2 (IT en RPT).

**Exp. 3 — Le SB decode-t-il quand on lui donne le SCH (3d), et que coutent les sondes (§4).**
```
CALYPSO_DSP_BUDGET=8000 CALYPSO_RIF_FCCH_ONLY=1 CALYPSO_INTM_TRANS=0 CALYPSO_CORRELATOR_TRACE=0 ./start-direct.sh --dsp --no-attach
grep -c 'SB 0x' /tmp/calypso/logs/osmocon.log ; grep 'SB 0x' /tmp/calypso/logs/osmocon.log | grep -vc 'BSIC=0'
awk '$4=="0x084e" && $3=="DSP>WR"' /tmp/calypso/logs/mailbox.log | awk '{print $8}' | sort | uniq -c   # valeurs de a_sch[3]
grep '\[tdma\] tick' /tmp/calypso/logs/qemu.log | tail -2         # dsp_insn_total / temps : insn/s sans les deux sondes
```
Tranche : plus de deux valeurs distinctes d'`a_sch[3]` et un `BSIC != 0` repete =
l'entree etait le probleme ; le rapport `dsp_insn_total / t_virt` compare a 0,43 M
donne le cout des deux sondes.

---

## 7. Reserves

- Je n'ai relance aucun run : les chiffres du jour (0,43 M insn/s, 8/16 trames par
  tick, 1,8 %/75 %, 404/280/124, pages, SCH, DMA) sont ceux fournis, pas rejoues.
- **`TDMA_REALTIME=1` comme mode vivant** est deduit de `calypso.env:59` et de l'ordre de
  chargement (`load.env:29-52`, `calypso.env` en dernier, `:=` n'ecrase qu'une valeur
  vide) ; je n'ai pas lu de `qemu-manifest.log` du jour. Si le manifeste dit VIRTUAL,
  la ligne « rattrapage » de 3a tombe mais pas le reste (budget < travail par trame).
- L'arithmetique « 16000 insn = 37 ms = 8 trames » suppose que le tick n'a pas d'autre
  cout que `c54x_run` ; `/tmp/tdma_profile.log` (`trx.c:2110`) le mesurerait.
- La piste 3b/2 (IT vectorisee pendant un `RPT`) est une **lecture du code**
  (`c54x.c:15550-15563` sans garde `rpt_active`) : je n'ai pas vu une trace qui la
  montre en train de se produire.
- 3c « deux copistes » : je n'ai pas identifie AR1/AR2 du bloc `0xb001` ; la cible
  `0x0586` est celle du modele (`trx.c:1387`), pas prouvee cote ROM.
- Le comptage « 60 variables mortes » ne voit que quatre idiomes de lecture ; au moins
  une (`CORRELATOR_TRACE`) est un faux positif, il peut y en avoir d'autres du meme type.
- Le desassembleur (`tools/tic54x-dis.py`) imprime les mots supplementaires comme des
  operandes (`ETAT_ACTUEL §12.5bis`) : j'ai lu les mots bruts (`76f8 08f8 0000`,
  `69f8 08f8 0001`) pour `0xb2cc` et `0x79e4`, pas le rendu ; `0xb214 mvdd` est lu
  sans ses operandes.
- `start-direct.sh` vit dans `/opt/GSM/osmo-operator`, hors de ce depot.
