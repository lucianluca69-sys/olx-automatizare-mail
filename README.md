# OLX Resell Finder

Automatizare n8n care scanează zilnic OLX.ro (electronice, IT și electrocasnice) pentru oportunități de resell profitabile, estimează profitul potențial cu AI și trimite dimineața un raport pe email cu linkul către un Google Sheet centralizat.

Workflow live: https://lucax12.app.n8n.cloud/workflow/3ZeC8PNOykKzk1Am

## Ce face

În fiecare zi la **08:00**:

1. Extrage conținutul (markdown) a 3 căutări OLX (Constanța ≤1500 lei, național ≤1500 lei, național 1501–5000 lei) folosind Firecrawl (scraping rezistent la pagini randate în JavaScript, prin operația „Scrape" — acoperită de Gateway credits).
2. Un nod AI (Claude Sonnet 5) citește markdown-ul fiecărei pagini și extrage lista structurată de anunțuri (titlu, preț, link, localitate, județ, categorie, stare) ca JSON.
3. Elimină anunțurile fără preț/link valid și pe cele deja trimise anterior (evidență ținută într-un Data Table n8n, `olx_anunturi_trimise`).
4. Pentru fiecare anunț nou, un AI Agent (Claude Sonnet 5, cu acces la Brave Search) estimează prețul de nou, prețul mediu OLX pentru produse similare, profitul estimat în lei și procentual, plus un motiv scurt în română.
5. Selectează 10–15 anunțuri (prioritate ~65% județul Constanța, restul din țară; maxim ~20% produse defecte/pentru piese, doar dacă profitul e foarte bun) și adaugă, dacă există, o oportunitate suplimentară până în 5000 lei.
6. Sortează totul descrescător după profitul estimat (%).
7. Salvează rândurile într-un Google Sheet și trimite un email cu topul 3 oportunități + linkul către Sheet complet.

## Arhitectură (WAT)

```
Trigger (zilnic 08:00)
  -> Setări (configurare editabilă: email, linkuri OLX, bugete)
  -> Pregătește URL-urile (Code: 1 item -> 3 itemi, câte unul per căutare)
  -> Extrage o pagină OLX (Firecrawl Scrape, format markdown; rulează o dată per URL)
  -> Extrage structurat cu AI (Claude Sonnet 5: markdown -> JSON cu anunțuri)
  -> Extrage anunțuri din pagini (Code: combină cele 3 rezultate într-o listă de anunțuri)
  -> Elimină anunțuri incomplete
  -> Elimină anunțuri deja trimise (Data Table dedup)
  -> Estimează profit anunț (AI Agent: Anthropic + Brave Search + output structurat)
  -> Combină date anunț
  -> Selectează și ordonează oportunitățile (Code: regiune, cotă defecte, oportunitate specială, sortare)
       -> Trimite raport email (Gmail)
       -> Desparte rândurile pentru salvare
            -> Salvează în Google Sheet
            -> Marchează anunțul ca trimis (Data Table)
```

Toate modelele AI (Anthropic Claude) și Brave Search, precum și scraping-ul Firecrawl, rulează pe **Gateway credits** din contul n8n — nu necesită chei API separate. Gmail și Google Sheets folosesc conturile Google deja conectate în n8n.

**Notă tehnică**: operația „Extract" a Firecrawl (extragere structurată automată) nu este acoperită de Gateway credits — doar „Scrape" (fetch simplu de conținut), „Map/Search" și „Crawl" sunt gratuite. De aceea extragerea structurată a anunțurilor este făcută separat, de un nod AI Anthropic care primește markdown-ul paginii și returnează JSON.

## Notă despre costuri (Gateway credits)

AI Agent-ul de estimare profit rulează pe **Gateway credits** din contul n8n. Pentru a limita costul, workflow-ul procesează cel mult **25 de anunțuri candidate** per rulare (nod „Limitează candidații"), suficient pentru cele 10-15 anunțuri finale din raport. La primul test complet, contul a rulat fără această limită și a epuizat creditele Gateway disponibile — dacă activarea zilnică eșuează cu eroarea „Payment required / Gateway credits depleted", e nevoie fie de suplimentarea creditelor din contul n8n, fie de o cheie API Anthropic/Brave proprie configurată direct pe nodurile respective.

## Configurare (finalizată automat)

Următorii pași au fost deja realizați:

- ✅ Data Table `olx_anunturi_trimise` creat (coloane `link_anunt`, `data_trimis`).
- ✅ Google Sheet „OLX Resell Finder - Raport zilnic" creat și legat de nodul „Salvează în Google Sheet".
- ✅ Linkul din email (`link_sheet` din nodul „Setări") actualizat cu URL-ul real al Sheet-ului.
- ✅ Cele 3 linkuri de căutare OLX verificate manual (returnează anunțuri reale, filtrele funcționează).

## Raport de date brute (trimis manual)

Pe 13.09.2026 a fost trimis pe email (lucianluca69@gmail.com) un raport cu cele 60 de anunțuri extrase cu succes de pe OLX în timpul testării (titlu, preț, localitate, categorie, stare, link) — fără estimarea AI a profitului, care nu a putut rula din lipsă de credite Gateway. Trimiterea s-a făcut printr-un workflow ajutător separat, `OLX Resell Finder - Trimite date brute (test)` (poate fi șters din n8n, a fost doar pentru acest raport punctual).

## Pași rămași înainte de activare

1. **Testează workflow-ul manual** (Execute Workflow în n8n) și verifică emailul primit + rândurile din Google Sheet.
2. **Activează workflow-ul** din n8n (toggle-ul Active) după ce testul confirmă că totul funcționează.

## Limitări cunoscute

- OLX nu are API public — scraping-ul (via Firecrawl) se poate rupe dacă OLX își schimbă structura site-ului; verifică din când în când.
- Estimările de preț ale AI-ului sunt aproximative (bazate pe căutări web), nu prețuri garantate — verifică manual înainte de a cumpăra orice produs.
- Extragerea structurată a anunțurilor depinde de calitatea markdown-ului livrat de Firecrawl și de acuratețea modelului AI — pot apărea ocazional anunțuri omise sau câmpuri greșit completate.
- Dacă într-o zi nu sunt suficiente anunțuri profitabile, raportul poate conține mai puțin de 10 anunțuri.
