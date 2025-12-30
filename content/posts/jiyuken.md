---
title: "じゆうけんきゅう : かいしこどんのまえにあるatgのかず"
date: 2019-06-28T18:28:00+09:00
tags: ["生物"]
---

真核生物の翻訳において、教科書的には「mRNAの5' GCapからMet tRNAが結合したリボソーム小サブユニットがmRNAをスキャンし、最初のAUGで停止して翻訳を開始する」とされるが、 <https://www.jstage.jst.go.jp/article/kagakutoseibutsu/54/3/54_191/_pdf> にあるように本体の遺伝子のより上流にAUGや終止コドンが存在する場合があるという。

そこで、NCBIの遺伝子データベースからヒトの第一染色体内の遺伝子の一部を自動で取得し、開始コドンの位置と開始コドンより前のAUGの数を計算するプログラムを作った。

```python

import requests
import time
import lxml.etree
import itertools
import re

# search record
def search(db, term, retmax, retstart):
    time.sleep(1)
    return requests.get("https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi", 
        params={"db" : db, "term" : term, "retmax" : retmax, "retstart" : retstart}).content

# fetch record independently
def fetch(db, key):
    time.sleep(1)
    return requests.get("https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi",
        params={"db" : db, "id" : key, "retmode" : "xml"}).content

# search and fetch records
def search_and_fetch(db, term, retmax, retstart):
    time.sleep(1)
    search_data = requests.get("https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi", 
        params={"db" : db, "term" : term, "retmax" : retmax, "retstart" : retstart, "usehistory" : "y"}).content
    tree = lxml.etree.XML(search_data)
    webenv = tree.find("WebEnv").text
    query_key = tree.find("QueryKey").text
    fetch_data = requests.get("https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi", 
        params={"db" : db, "retmax" : retmax, "retstart" : retstart, "query_key" : query_key, "WebEnv" : webenv, "retmode" : "xml"}).content
    return fetch_data

# get all transcription variants and corresponding peptides
#    from an Entrezgene XML element
# returns : list of mrna IDs, list of list of peptide IDs
def get_all_transcription_variants(entrezgene):
    mrna_list = []
    protein_list_list = []
    for mRNA in entrezgene.find("Entrezgene_locus").findall("Gene-commentary/Gene-commentary_products/"):
        protein_list = []
        for peptide in mRNA.findall("Gene-commentary_products/"):
            accession = peptide.find("Gene-commentary_accession")
            if accession == None:
                continue
            protein_list.append(accession.text)
        mrna_list.append(mRNA.find("Gene-commentary_accession").text)
        protein_list_list.append(protein_list)
    return mrna_list, protein_list_list

# search gene ids
def search_gene_ids(query, num=100, start=0):
    index_data = search("gene", query, num, start)

    tree = lxml.etree.XML(index_data)

    return list(map(lambda x: x.text, index_data.findall("IdList/")))    

# get all gene data from search result
def get_all_gene_data(term, retmax, retstart):
    result = search_and_fetch("gene", term, retmax, retstart)
    mrna_list = []
    for gene in lxml.etree.XML(result).findall("Entrezgene"):
        mrna_list += get_all_transcription_variants(gene)[0]
    nuccore_data = fetch("nuccore", ",".join(mrna_list))
    tree = lxml.etree.XML(nuccore_data)
    mrna_list = []
    for mrna in tree.findall('GBSeq'):
        mrna_id = mrna.find("GBSeq_locus").text
        transcription = mrna.find("GBSeq_sequence").text
        peptide_list = []
        for label in mrna.xpath('GBSeq_feature-table/GBFeature/GBFeature_quals/GBQualifier/GBQualifier_name[text()="translation"]'):
            protein_id = label.xpath('../../GBQualifier/GBQualifier_name[text()="protein_id"]')[0].find("../GBQualifier_value").text
            translation = label.find("../GBQualifier_value").text
            peptide_list.append([protein_id, translation])
        mrna_list.append([mrna_id, transcription, peptide_list])
    return mrna_list

codon_table = dict(zip(
    map("".join, [a + b + c for a in "tcag" for b in "tcag" for c in "tcag"]),
    'FFLLSSSSYY**CC*WLLLLPPPPHHQQRRRRIIIMTTTTNNKKSSRRVVVVAAAADDEEGGGG'))

inv_codon_table = {}
for k, v in codon_table.items():
    if v not in inv_codon_table:
        inv_codon_table[v] = [k]
    else:
        inv_codon_table[v] += [k]

def reverse_translate_regex(peptide):
    return "".join(map(lambda x: "(" + "|".join(inv_codon_table[x]) + ")", peptide))

def get_initiation_pos(rna, peptide):
    return re.search(reverse_translate_regex(peptide), rna).regs[0][0] + 1

genedata = get_all_gene_data("NC_000001.11[Nucleotide Accession]", 10, 200)

positions = []

for rna in genedata:
    rna_id = rna[0]
    rna_sequence = rna[1]
    for peptide in rna[2]:
        peptide_id = peptide[0]
        peptide_sequence = peptide[1]
        try:
            pos = get_initiation_pos(rna_sequence, peptide_sequence)
        except:
            pos = None
        try:
            num_of_atgs_before_init = len(re.findall("atg", rna_sequence[:pos - 1]))
        except:
            num_of_atgs_before_init = None
        positions.append([rna_id, peptide_id, pos, num_of_atgs_before_init])

print(positions)

```

Output(pprint) : mRNA id, peptide id, 開始コドンの位置, 開始コドンより前のAUG個数

```python
[['NM_001146068', 'NP_001139540.1', 237, 3],
 ['NM_001748', 'NP_001739.3', 103, 0],
 ['XM_017002189', 'XP_016857678.1', 237, 2],
 ['NM_001290403', 'NP_001277332.1', 327, 3],
 ['NM_003189', 'NP_003180.1', 445, 3],
 ['XM_017002193', 'XP_016857682.1', 271, 3],
 ['XM_017002191', 'XP_016857680.1', 58, 1],
 ['XM_017002190', 'XP_016857679.1', 933, 8],
 ['NM_001290405', 'NP_001277334.1', 200, 2],
 ['NM_001290404', 'NP_001277333.1', 332, 3],
 ['NM_001287347', 'NP_001274276.1', 217, 1],
 ['XM_017002188', 'XP_016857677.1', 434, 4],
 ['XM_017002187', 'XP_016857676.1', 1242, 11],
 ['XM_005271160', 'XP_005271217.1', 1827, 13],
 ['NM_001290406', 'NP_001277335.1', 91, 2],
 ['XM_017002192', 'XP_016857681.1', 421, 4],
 ['XM_017002027', 'XP_016857516.1', 543, 8],
 ['NM_000329', 'NP_000320.1', 50, 0],
 ['NM_000329', 'NP_000320.1', 50, 0],
 ['NM_004905', 'NP_004896.1', 64, 0],
 ['NM_203342', 'NP_976217.1', 835, 5],
 ['NM_001166005', 'NP_001159477.1', 128, 0],
 ['NM_203343', 'NP_976218.1', 119, 0],
 ['NM_004437', 'NP_004428.1', 809, 4],
 ['NM_001166007', 'NP_001159479.1', 729, 3],
 ['XM_011540964', 'XP_011539266.1', 726, 3],
 ['XM_017000595', 'XP_016856084.1', 726, 3],
 ['XM_017000594', 'XP_016856083.1', 726, 3],
 ['XM_017000596', 'XP_016856085.1', 726, 3],
 ['XM_017000593', 'XP_016856082.1', 726, 3],
 ['XM_017000597', 'XP_016856086.1', 726, 3],
 ['XM_024453880', 'XP_024309648.1', 726, 3],
 ['XM_017000599', 'XP_016856088.1', 726, 3],
 ['XM_017000598', 'XP_016856087.1', 726, 3],
 ['XM_017000600', 'XP_016856089.1', 726, 3],
 ['XM_017000603', 'XP_016856092.1', 726, 3],
 ['XM_017000602', 'XP_016856091.1', 726, 3],
 ['XM_017000604', 'XP_016856093.1', 726, 3],
 ['XM_017000584', 'XP_016856073.1', 495, 7],
 ['XM_017000585', 'XP_016856074.1', 492, 7],
 ['XM_011540956', 'XP_011539258.1', 203, 0],
 ['XM_011540958', 'XP_011539260.1', 203, 0],
 ['XM_005245760', 'XP_005245817.1', 203, 0],
 ['XM_006710434', 'XP_006710497.1', 203, 0],
 ['XM_006710439', 'XP_006710502.1', 203, 0],
 ['XM_005245753', 'XP_005245810.1', 203, 0],
 ['XM_005245761', 'XP_005245818.1', 203, 0],
 ['XM_005245768', 'XP_005245825.1', 203, 0],
 ['XM_011540959', 'XP_011539261.1', 203, 0],
 ['XM_005245763', 'XP_005245820.1', 203, 0],
 ['XM_011540961', 'XP_011539263.1', 203, 0],
 ['XM_017000586', 'XP_016856075.1', 203, 0],
 ['XM_005245757', 'XP_005245814.1', 203, 0],
 ['XM_005245765', 'XP_005245822.1', 203, 0],
 ['XM_011540960', 'XP_011539262.1', 203, 0],
 ['XM_005245764', 'XP_005245821.1', 203, 0],
 ['XM_011540962', 'XP_011539264.1', 203, 0],
 ['XM_005245769', 'XP_005245826.1', 203, 0],
 ['XM_005245774', 'XP_005245831.1', 203, 0],
 ['XM_011540957', 'XP_011539259.1', 203, 0],
 ['XM_017000583', 'XP_016856072.1', 203, 0],
 ['XM_005245770', 'XP_005245827.1', 203, 0],
 ['XM_017000581', 'XP_016856070.1', 203, 0],
 ['XM_017000589', 'XP_016856078.1', 203, 0],
 ['XM_017000582', 'XP_016856071.1', 203, 0],
 ['XM_017000590', 'XP_016856079.1', 203, 0],
 ['XM_005245772', 'XP_005245829.1', 203, 0],
 ['XM_011540963', 'XP_011539265.1', 203, 0],
 ['XM_011540965', 'XP_011539267.1', 203, 0],
 ['XM_005245773', 'XP_005245830.1', 203, 0],
 ['XM_017000591', 'XP_016856080.1', 203, 0],
 ['XM_017000587', 'XP_016856076.1', 203, 0],
 ['XM_017000588', 'XP_016856077.1', 203, 0],
 ['XM_017000592', 'XP_016856081.1', 203, 0],
 ['NM_001166006', 'NP_001159478.1', 201, 0],
 ['NM_203342', 'NP_976217.1', 835, 5],
 ['NM_006610', 'NP_006601.2', 33, 0],
 ['XM_017000097', 'XP_016855586.1', 33, 0],
 ['NM_139208', 'NP_631947.1', 33, 0],
 ['NM_006610', 'NP_006601.2', 33, 0],
 ['NM_139208', 'NP_631947.1', 33, 0],
 ['NM_004759', 'NP_004750.1', 326, 0],
 ['NM_032960', 'NP_116584.2', 326, 0],
 ['XM_005273353', 'XP_005273410.1', 319, 0],
 ['XM_017002810', 'XP_016858299.1', 156, 0],
 ['NM_032960', 'NP_116584.2', 326, 0],
 ['NM_001127651', 'NP_001121123.1', 106, 0],
 ['XM_011509580', 'XP_011507882.1', 114, 0],
 ['NM_001190789', 'NP_001177718.1', 276, 3],
 ['NM_001190794', 'NP_001177723.1', 276, 3],
 ['NM_000433', 'NP_000424.2', 276, 3],
 ['XM_011509581', 'XP_011507883.1', 101, 2],
 ['XM_005245207', 'XP_005245264.1', 205, 1],
 ['NM_000433', 'NP_000424.2', 276, 3],
 ['NM_001514', 'NP_001505.1', 69, 0],
 ['XM_011541299', 'XP_011539601.1', 415, 6]]
```

結果としては、かなり多くの場合で開始コドンよりまえにAUGがあることがわかった。

