# Pravidlo: Meta kampaně — UTM, pojmenování a GA4 konektor (všichni klienti)

Toto pravidlo platí vždy, když zakládáš, duplikuješ, upravuješ nebo spouštíš kampaně, sady reklam nebo reklamy v Meta Ads (přes CLI, MCP, Marketing API nebo když připravuješ zadání pro ruční nasazení). Platí pro Pudeto i pro každého klienta. Cíl: každá návštěva z Mety se v GA4 napáruje na náklad kampaně importovaný Meta konektorem. Bez toho GA4 zobrazí náklady a výsledky na dvou různých řádcích a PNO po kampaních nejde spočítat.

## 0. Jak GA4 páruje Meta náklady (proč je pravidlo přesně takové)

GA4 importuje náklady z Mety přes konektor **Admin → Import dat → Meta** („Údaje o kampaních"). Konektor má pevně nastavené hodnoty, na které GA4 páruje:

| Pole konektoru | Hodnota |
|---|---|
| Facebook source (`utm_source`) | `fb` |
| Instagram source (`utm_source`) | `ig` |
| Audience Network source (`utm_source`) | `an` |
| Messenger source (`utm_source`) | `msg` |
| Medium (`utm_medium`) | `cpc` |
| klíč spojení | `utm_id` = ID kampaně v Metě |

Referenční nastavení Pudeto: Business Center *Pudeto*, Ad Account `1679873289154726`, import denně 04:00–05:00. U jiného klienta je to jeho GA4 property a jeho Ad Account, ale hodnoty zdrojů a media musí být stejné.

Meta parametr `{{site_source_name}}` vrací přesně `fb` / `ig` / `an` / `msg`, proto je v šabloně. Jakákoli jiná hodnota `utm_source` nebo `utm_medium` (např. `facebook`, `paid-social`, `instagram`, `social`) nebo chybějící `utm_id` = rozpojený join. Konektor a šablona se musí shodovat; pokud u nějaké property nesouhlasí, je to blokující problém, který se řeší PŘED spuštěním, ne po něm.

## 1. URL parametry — jediná povolená hodnota

Každá reklama s cílovou URL má v poli `url_tags` (Ads Manager: sekce Tracking → „URL parameters") přesně tento řetězec, beze změny:

```
utm_source={{site_source_name}}&utm_medium=cpc&utm_campaign={{campaign.name}}&utm_id={{campaign.id}}&utm_content={{adset.name}}&utm_term={{ad.name}}
```

- Platí pro všechna umístění včetně Instagramu, Messengeru a Audience Network — o zdroj se stará `{{site_source_name}}`, ručně se nikdy nevpisuje.
- Do cílové URL (`link` / `object_story_spec`) se UTM nikdy nepíší. Pokud v URL nějaké najdeš, odstraň je a přesuň do `url_tags`.
- Používej dynamické parametry (`{{campaign.name}}` atd.), nikdy ručně vepsaný název.
- Šablona je jedna pro všechny klienty. Nepřijímej „klientskou variantu" `utm_source/utm_medium` — místo toho se u klienta nastaví GA4 konektor na hodnoty z bodu 0.

Historická poznámka: do 22. 9. 2026 platila šablona `utm_source=facebook&utm_medium=paid-social&...&utm_content={{adset.name}}_{{ad.name}}&utm_term={{placement}}`. Ta se s GA4 konektorem nepáruje. Reklamy, které ji ještě nesou, se musí přestavět (nová kreativa + swap, bod 8).

## 2. Název kampaně

Schéma (bez diakritiky, bez mezer, velká písmena, podtržítka):

```
{KLIENT}_{LOKALITA}_{TYP}_{FAZE}
```

- `KLIENT` — kód klienta (např. `BF27` = Bezva Fest 2027).
- `LOKALITA` — trh nebo město; `ALL` = celoplošné. U Bezva Festu: `PRG`, `HK`, `PLZ`, `SLA`, `LTM`, `ALL`.
- `TYP` — `PROSPECT`, `RETARGET`, `CTWA`, `LEAD`, `CATALOG`, `BRAND`.
- `FAZE` — volný kód fáze; u Bezva Festu `V1`–`V5` podle cenové vlny, `LAST` před akcí.

Validační regex, který název musí splnit před zápisem:

```
^[A-Z0-9]{2,8}_[A-Z0-9]{2,6}_(PROSPECT|RETARGET|CTWA|LEAD|CATALOG|BRAND)_[A-Z0-9]{1,8}$
```

Sady reklam a reklamy: popisný název bez mezer (publikum, kreativa, formát). Nesmí obsahovat `&`, `=`, `?`, `#`, `/` — rozbily by UTM. Název sady jde do `utm_content`, název reklamy do `utm_term`, takže obojí má být čitelné i v GA4 reportu.

## 3. Co nikdy nedělat

1. **Nepřejmenovávat** spuštěnou kampaň. Změna názvu rozpojí historické návštěvy od nákladů. Potřebuješ-li změnu, duplikuj pod novým názvem a starou pozastav.
2. **Neměnit** `utm_source` ani `utm_medium`, ani u jediné reklamy — jsou dané konektorem GA4.
3. **Nezakládat** kampaň, jejíž název neprojde regexem, ani reklamu bez `url_tags`. Místo zápisu vrať chybu a navrhni opravu.
4. **Nespouštět** (ACTIVE) kampaň, u které neproběhla kontrola z bodu 5 a uživatel nepotvrdil nastavení GA4 konektoru.

## 4. Postup před každým zápisem

1. Zkontroluj název proti regexu.
2. Zkontroluj, že `url_tags` je přesně šablona z bodu 1 (porovnej řetězec, ne „přibližně").
3. Zkontroluj, že cílová URL neobsahuje `utm_`.
4. Pokud upravuješ existující reklamy, nejdřív vypiš ty, které šablonu nesplňují, a navrhni opravu — sám je nepřepisuj bez schválení.
5. Zápis proveď až po lidském schválení (návrh → kontrola → schválení → provedení → auditní záznam).
6. V auditním záznamu uveď ID kampaně, název a použitý `url_tags`.

## 5. Kontrola před spuštěním (GA4 brána) — POVINNÁ, nikdy nepřeskakovat

Před každou aktivací kampaně (nebo před předáním kampaně k ruční aktivaci) proveď a vypiš:

1. **Vysvětli uživateli, proč to děláš** — pokaždé, i když to už slyšel: GA4 páruje Meta náklady na `utm_id` = ID kampaně a na `utm_source`/`utm_medium` rovné hodnotám konektoru (fb/ig/an/msg + cpc). Jiné hodnoty = náklad a návštěvy na dvou řádcích, PNO po kampaních nelze spočítat.
2. **Důrazně se zeptej**, zda je GA4 konektor Meta u dané property nastaven podle bodu 0: správný Ad Account, zdroje `fb` / `ig` / `an` / `msg`, medium `cpc`, import běží, autorizace není expirovaná. Ty do GA4 nevidíš — potřebuješ potvrzení uživatele (ideálně screenshot detailu zdroje dat). Bez potvrzení se nespouští. Pokud konektor má jiné hodnoty, STOP: nejdřív se sladí konektor (preferováno) a teprve pak se spouští.
3. **Audit `url_tags`** všech reklam v kampani skriptem z bodu 8: každá reklama musí mít šablonu doslova, žádný link nesmí obsahovat `utm_`. Nesplňující vypiš a navrhni opravu.
4. **Validuj název kampaně** regexem z bodu 2.
5. Teprve po čtyřech zelených bodech a výslovném pokynu uživatele: aktivace (dry-run → `--confirm`).

## 6. Kontrola po spuštění

Do 24 hodin od spuštění: v GA4 (Reports → Acquisition → Traffic acquisition, dimenze Session campaign) musí mít řádek s názvem kampaně současně návštěvy i náklad. Řádek s nákladem a nulovými návštěvami = rozpojený join, opravit `url_tags` nebo konektor.

## 7. Organický obsah (není reklama)

Odkazy v biu, ve stories bez placení, v organických příspěvcích:

```
?utm_source=instagram&utm_medium=social&utm_campaign=bio
?utm_source=facebook&utm_medium=social&utm_campaign=post
```

## 8. Bez cílové URL

Click-to-WhatsApp, Messenger, lead formuláře, propagace profilu: nemají web, UTM se neuplatní. Výkon se měří v Metě a v cílovém systému (WhatsApp, CRM). Kampaň přesto dodržuje schéma názvu.

## 9. Mapování na meta-ads-app CLI

| Krok pravidla | Příkaz / postup |
|---|---|
| Zápis `url_tags` při tvorbě kreativy | `creative-create ... --url-tags '<šablona z bodu 1>'` — flag je POVINNÝ u každé kreativy s `--link` |
| Klon Advantage+ kreativy | `creative-clone` přenáší `url_tags` ze zdroje automaticky; zkontroluj výstup dry-runu — pokud zdroj nese starou šablonu (`facebook/paid-social`), přepiš `--url-tags` na aktuální šablonu |
| Boost organiky (`creative-from-post`, `creative-from-ig`) | CLI nemá `--url-tags`. Kreativa vzniká bez UTM → po vytvoření ověř `creative-detail`; pokud má kreativa cílový link, upozorni uživatele, že UTM chybí a je nutné je doplnit ručně v Ads Manageru (Tracking → URL parameters) nebo postavit kreativu přes `creative-create`. Bez cílové URL (čistý boost postu) se UTM neuplatní — bod 8 |
| Audit existujících reklam | výpis `ads` UTM nevrací; `url_tags` je jen v `ad-detail --ad-id ... --json` (`creative.url_tags`) a `creative-detail`. Projdi reklamy sekvenčně (rate limit), porovnej se šablonou přesným string-equal, nesplňující vypiš a navrhni opravu (creative je immutable → nová kreativa + swap přes `creative-clone --url-tags ... --swap-on-ad`, po swapu běží re-review) |
| Kontrola cílové URL | v `ad-detail --json` / `creative-detail` zkontroluj `creative.link_url`, `object_story_spec.link_data.link` a `asset_feed_spec.link_urls[].website_url`, že neobsahují `utm_` |
| Validace názvu kampaně | před `campaign-create --name` / `campaign-duplicate` ověř regex z bodu 2; `campaign-update --name` na spuštěné kampani nikdy (bod 3.1) |
| Aktivace | `campaign-update --status ACTIVE --confirm` jen po bráně z bodu 5 a na výslovný pokyn |
| Auditní záznam | po `--confirm` vypiš ID kampaně, název a přesný `url_tags` |

Kontrolní skript pro audit (dosaď ID kampaně; sekvenčně, bez paralelizace):

```bash
CLI=/Users/martindlouhy/Projects/nastroje/meta-ads-app/run.sh
TEMPLATE='utm_source={{site_source_name}}&utm_medium=cpc&utm_campaign={{campaign.name}}&utm_id={{campaign.id}}&utm_content={{adset.name}}&utm_term={{ad.name}}'
for id in $($CLI ads --campaign-id <ID> --json | jq -r '.[].id'); do
  $CLI ad-detail --ad-id "$id" --json \
    | jq -r --arg t "$TEMPLATE" '[.id, .name, (if (.creative.url_tags // "") == $t then "OK" else (.creative.url_tags // "MISSING") end), (.creative.link_url // "-")] | @tsv'
done
```

Řádek s `MISSING` nebo s jiným řetězcem než `OK` = nesplňuje pravidlo. Link obsahující `utm_` = nesplňuje pravidlo.
