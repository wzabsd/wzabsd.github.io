Cookbook for trans-omics analysis in R
================

This is an [R Markdown](http://rmarkdown.rstudio.com) Notebook. When you
execute code within the notebook, the results appear beneath the code.

Try executing this chunk by clicking the *Run* button within the chunk
or by placing your cursor inside it and pressing *Ctrl+Shift+Enter*.

Add a new chunk by clicking the *Insert Chunk* button on the toolbar or
by pressing *Ctrl+Alt+I*.

When you save the notebook, an HTML file containing the code and output
will be saved alongside it (click the *Preview* button or press
*Ctrl+Shift+K* to preview the HTML file).

The preview shows you a rendered HTML copy of the contents of the
editor. Consequently, unlike *Knit*, *Preview* does not run any R code
chunks. Instead, the output of the chunk when it was last run in the
editor is displayed.

![](/home/wzabsd/Desktop/TransomicAnalysis/Transomic_training/Trans-omics%20analysis.png)

# Data preparing

-   Since the raw data is in metalab format, using matlab
    function`writematrix(A,filename)` to convert it into csv file.
-   Add col and row name based on given label in matlab file.

## 1. Import omics data

``` r
data_trans = read_csv("yourdata") #change parameter based on your requirements
data_metab=read_csv("yourdata") # imported data is in tibble format
data=base::as.data.frame(data)#If you do not like tibble format dataset
View(data_trans)
View(data_metab)
```

## 2. Responsive molecules identification

Remove those are absent in more than half of replicates molecules

``` r
trans_sample_num=list(1:11,12:14,15:17,18:20,21:23,24:35,36:38,39:41,42:44,45:47)
metab_sample_num=list(1:5,6:10,11:15,16:20,21:25,26:30,31:35,36:40,41:45,46:50)
data_trans[is.na(data_trans)]=0 #replace nan data with 0
data_metab[is.na(data_metab)]=0

missing_filter=function(df,group){
  output=c()
  for (i in 1:dim(df)[1]) {
    for (j in group) {
      if (median(as.matrix(df[i,j]))==0) {
        output=c(output,row.names(df)[i])
        break
      }
    }
  }
  return(unique(output))
}

del_miss_trans=missing_filter(data_trans,trans_sample_num)
#del_miss_metab=missing_filter(data_metab,metab_sample_num)
data_trans_=data_trans[!(row.names(data_trans)%in%del_miss_trans),]
#data_metab_=data_metab[!(row.names(data_metab)%in%del_miss_metab),]
```

Calculate FC, p value and p-adjust value

``` r
#An example,this function can be written better...
calculator=function(df,group_name,con_group,exp_group){
  p_value=c()
  FC=(rowSums(df[,exp_group[[1]]])/length(exp_group[[1]]))/(rowSums(df[,con_group[[1]]])/length(con_group[[1]]))
  for (i in 1:dim(df)[1]) {
  p_value=c(p_value,t.test(df[i,exp_group[[1]]],df[i,con_group[[1]]])$p.value)
  }
  p_value_adjust=p.adjust(p_value,method="BH")
  colname=c(base::paste('FC_',group_name,sep=''),base::paste('p_',group_name,sep=''),base::paste('p_',group_name,'_adjust',sep=''))
  df[,c(colname[1])]=FC
  df[,c(colname[2])]=p_value
  df[,c(colname[3])]=p_value_adjust
  df#return new dataframe
}

#group_name=c(c('wt20','wt60','wt120','wt240'),c('ob20','ob60','ob120','ob240'))
#group_position=c(2:5,7:10)
data_trans_=calculator(data_trans_,'wt20',trans_sample_num[1],trans_sample_num[2])
```

FC>1.5 or \<0.67 and p-adjust value \<0.1

``` r
trans_data_position=list(48:50)#list(48:50,51:53,54:56,57:59)
FC_filter=function(df,group){
  output=c()
  for (i in 1:dim(df)[1]) {
    for (j in group) {
      if ((df[i,j[1]]>1.5|df[i,j[1]]<0.67)&df[i,j[3]]<0.1) {
        output=c(output,row.names(df)[i])
        break
      }
    }
  }
  unique(output)
}
response_trans=FC_filter(data_trans_,trans_data_position)#response transcripts
```

# Information retrieving

## 1. ID conversion

From ensemble to gene ID (NCBI database).

``` r
library(biomaRt)
ensembl=useEnsembl(biomart = 'genes',dataset = 'mmusculus_gene_ensembl',mirror = 'asia')
trans_info=getBM(attributes = c('ensembl_gene_id','entrezgene_id', 'transcript_length'),
      values = response_trans, 
      filters = 'ensembl_gene_id',
      mart = ensembl)
```

Remove duplicated and NA data.

``` r
#Clean dirty retrieve results(base on length)
trans_info=trans_info%>%drop_na()
trans_info=trans_info[order(trans_info$transcript_length,decreasing = TRUE),]
trans_info=trans_info%>%distinct(entrezgene_id,.keep_all = TRUE)
length(unique(trans_info$ensembl_gene_id))#There are some transcripts having multiple gene_ID retrieve results, but can be partially filtered based on EC_number retrieve.
trans_info[duplicated(trans_info$ensembl_gene_id),]
```

## 2. Retrieve information from various database

Most of these package are downloaded from bioconductor. Please read
manual guidance or toy example if you want:)

### 1. Retrieve information from KEGG

#### Get KEGG_id

Basically the gene_id = kegg_id, but there are still some exceptions???

``` r
library(KEGGREST)
#Decrease 403 possibility...
testit =function(x)
{
    p1= proc.time()
    Sys.sleep(x)
    proc.time() - p1 # The cpu usage should be negligible
}
for (i in 1:dim(trans_info)[1]) {
  query=keggConv('mmu',paste('ncbi-geneid:',trans_info$entrezgene_id[i],sep=''))
  if (is_empty(query)==FALSE) { #you need to import tidyverse
    trans_info$kegg_id[i]=query
  }else{
    trans_info$kegg_id[i]='Not match' #There are also some gene_id cannot be mapped to kegg_id
    print(trans_info$entrezgene_id[i])
  }
  testit(0.1) #set a 0.1s length interval,experiential number
}
```

#### Get EC_number

``` r
trans_info_refine=subset(trans_info,kegg_id!='Not match')# Remove not matched data
for (i in 1:dim(trans_info_refine)[1]) {
  query=keggGet(trans_info_refine$kegg_id[i])
  if (is.null(query[[1]]$ORTHOLOGY)) {
    trans_info_refine$orthology[i]=NA
  }else{
    trans_info_refine$orthology[i]=query[[1]]$ORTHOLOGY
  }
  testit(0.1)
}
```

#### Extract EC_number and refine

``` r
enzyme_info=trans_info_refine%>%drop_na()
enzyme_info[str_detect(enzyme_info$orthology,'EC:'),]#Remove non-enzyme data
invalid_enzyme=c()#Remove invalid EC_number results
for (i in 1:dim(enzyme_info)[1]) {
  EC=enzyme_info$orthology[i]%>%str_extract_all('(?<=\\[).+?(?=\\])')
  EC=EC[[1]]
  EC=EC[str_detect(EC,'EC')] #Remove other character enclosed in []
  EC=gsub('EC:','',EC) #Remove the "EC:" character
  EC=str_split(EC,' ')[[1]]
  EC_refine=EC[!str_detect(EC,'-')]
  enzyme_info$EC_number[i]=list(EC)
  if (length(EC_refine)>0) {
    enzyme_info_refine$EC_number[i]=list(EC_refine)
  }else{
    invalid_enzyme=c(invalid_enzyme,enzyme_info$orthology[i])
  }
}
enzyme_info_refine=enzyme_info[!(enzyme_info$orthology%in%invalid_enzyme),]
```

#### Get reaction, product and substrate information

``` r
parse_kegg=function(object,output){
  pattern=if_else(object=='reaction','(?<=\\[RN:).+?(?=\\])','(?<=\\[CPD:).+?(?=\\])')
  object=toupper(object)
  query_result=query[[object]]%>%str_extract_all(pattern=pattern)
  if (!is_empty(query_result)) {
    for (i in query_result) {
      if (is_empty(i)) {
        next
      }
      if (str_detect(i,' ')) { #Sometimes one reaction has multiple reaction_id
        for (j in str_split(i,' ')[[1]]) {
          output=c(output,j)
        }
      }else{
         output=c(output,i)
      }
    }
  }
  output
}
for (i in 1:dim(enzyme_info_refine)[1]) {
  reaction=c()
  product=c()
  substrate=c()
  for (j in enzyme_info_refine$EC_number[i]) {
    for (k in j) {
      query=keggGet(k)[[1]]
      reaction=parse_kegg('reaction',reaction)
      substrate=parse_kegg('substrate',substrate)
      product=parse_kegg('product',product) 
    }
  }
  enzyme_info_refine$Reaction[i]=list(unique(reaction))
  enzyme_info_refine$Product[i]=list(unique(product))
  enzyme_info_refine$Substrate[i]=list(unique(substrate))
}
```

### 2. Retrieve activator and inhibitor data from brenda(offline)

``` r
library(brendaDb)
brenda.filepath=DownloadBrenda()#Download brenda offline data in local cache directory
df_brenda=ReadBrenda(brenda.filepath)# Load brenda data as a tibble
```

#### 1. Get activator and inhibitor information

**Final workflow!!**

The below ways are just my trials??? At least they show some of my
thinking history???

1.  Get inhibitor and activator **result file** one-by-one from
    brenda(ligand_id included)

``` python
import requests
def brenda_downloader(EC_number,role,organisms):
    l=[]
    opsin='https://www.brenda-enzymes.org/result_download.php?a=13&RN=&RNV=&os=&pt=&FNV=1&tt=&SYN=&Textmining=&W[1]={0}&T[1]=2&V[1]=1&W[2]={1}&T[2]=2&V[2]=1&W[3]=&T[3]=2&W[4]={2}&T[4]=1&W[6]=&T[6]=2&W[9]=&T[9]=2&nolimit=1'
    headers={'Host': 'www.brenda-enzymes.org','User-Agent': 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.101 Safari/537.36 Edg/92.0.902.49'}
    organisms=organisms.replace(" ",'+')
    url=opsin.format(EC_number,role,organisms)
    response=requests.get(url,headers=headers)
    result=response.text
    for i in result.split("\n"):
        if len(i)>0:
          l.append(i.split("\t")[5])
    return l
ligand_list=brenda_downloader('3.4.24.17','Inhibitor','Mus musculus')
print(ligand_list)
```

    ## ['62', '696', '53037', '53038', '53039', '53040', '53041', '53042', '53043', '53044', '53045', '53046', '53047', '53048', '53049', '53050', '53051', '53052', '53053', '53054', '53055', '53056', '53057', '53058', '53059', '53060', '53061', '53062', '53063', '53064', '53065', '53066', '53067', '53068', '53069', '53070', '53071', '53072', '53073', '53074', '53075', '53076', '53077', '54397', '54398', '55208', '55209', '55210', '117258', '117259', '117260', '206294']

2.  Retrieve compound_id from UniChem with brenda ligand_id.

    And about more web services in Unichem, please read this
    [guideline](https://www.ebi.ac.uk/unichem/info/webservices).

``` python
import requests
def converter(brenda_id):
  opsin='https://www.ebi.ac.uk/unichem/rest/src_compound_id/{0}/{1}/{2}'
  src_id=37 #brenda index, and it can be changed to meed your needs
  to_src_id=6 #kegg index
  url=opsin.format(brenda_id,src_id,to_src_id)
  response = requests.get(url)
  result=response.json()[0] #type(result)==dict
  return result['src_compound_id']
kegg_id=converter(153)
print(kegg_id)
```

    ## C00097

#### 2. Other ways to get the brenda information.

Workflow_1 design:

<span style="color:red">**This method is hard to link brenda data with
compound_id**</span>

(So why I wrote this one???)

``` r
for (i in 1:dim(enzyme_info_refine)[1]) {
  activator=c()
  inhibitor=c()
  for (j in enzyme_info_refine$EC_number[i]) {
    for (k in j) {
      res=QueryBrenda(df_brenda,EC=k,fields =c("INHIBITORS",'ACTIVATING_COMPOUND'),organisms='Mus musculus')
      activator_result=res[[k]]$interactions$activating.compound
      inhibitor_result=res[[k]]$interactions$inhibitors
      if (is_empty(res[[k]]$interactions)|is_empty(dim(activator_result))|is_empty(dim(inhibitor_result))) {
        next
      }
      if (dim(activator_result)[1]>0) {
        activator=c(activator,activator_result$description)
      }
      if (dim(inhibitor_result)[1]>0) {
        inhibitor=c(inhibitor,inhibitor_result$description)
      }
    }
  }
  enzyme_info_refine$activator[i]=list(unique(activator))
  enzyme_info_refine$inhibitor[i]=list(unique(inhibitor))
}
#And then try to link these compound name with kegg compound_id...
```

Workflow_2 design:

1.  Get inhibitor and activator **result file** one-by-one from
    brenda(inchi included)

2.  Get **inchikey** by NIH CADD group [Chemical Identifier
    Resolver](https://cactus.nci.nih.gov/).

    ``` python
    import requests
    opsin = 'https://cactus.nci.nih.gov/chemical/structure/{0}/{1}'
    ide = 'InChI=1S/C3H7NO2S/c4-2(1-7)3(5)6/h2,7H,1,4H2,(H,5,6)/t2-/m0/s1'  # InChI
    ide = ide.replace('#', '%23')# '#' often exists in SMILES
    rep = 'stdinchikey'  # the desired output is StdInChIKey

    # for more representations
    #rep = 'smiles'      # the desired output is SMILES
    #rep = 'stdinchi'    # the desired output is StdInChI
    #rep = 'iupac_name'  # the desired output is IUPAC name
    #rep = 'cas'         # the desired output is CAS Registry Number
    #rep = 'formula'     # the desired output is Chemical Formula

    url = opsin.format(ide, rep)
    response = requests.get(url)
    #response.raise_for_status()
    result=response.text
    print(result)
    ```

        ## InChIKey=XUJNEKJLAYXESH-REOHCLBHSA-N

3.  Get compound_ID from Unichem.

    ``` python
    from chembl_webresource_client.unichem import unichem_client as unichem
    ret = unichem.get('XUJNEKJLAYXESH-REOHCLBHSA-N')
    for i in ret:
      if i['src_id']=='6':
        print(i['src_compound_id'])
    ```

        ## C00097

Workflow_3 design:

1.  Get all of the metabolic-pathway-related metabolites.
2.  Find intersection with brenda regulator result.

Workflow_4 design:

Just use given brenda data???

1.  Parse text file into csv.
2.  Just match.

### 3. Get TF information

TF data parsing from given file. And import the result file in Rstudio.

``` python
FileName = "yourpath"
f = open(FileName,'r',encoding='utf-8')
AllLines = f.readlines()
f.close()
ENSMUSG=''
f1 = open("yourpath", "w") #delimiter is '|'
for EachLine in  AllLines:
    EachLine=EachLine.strip()
    if len(EachLine)==0:
        continue
    if 'ENSMUSG' in EachLine:
        index=EachLine.find('ENSMUSG')
        ENSMUSG=EachLine[index:index+18]
        print(ENSMUSG)
        continue
    if EachLine[0]=='V':
        f1.write(EachLine[2:]+'|'+ENSMUSG+'\n')
f1.close()
```

Maybe you want to get new TF annotation by yourself:

1.  Get `GRCh38.gene.bed` and `GRCh38.TFmotif_binding.bed`(Just like
    these) from ensemble(biomart), and note that not all version of that
    database has binding motif file, so maybe you must change its
    version.

    (I think these information also can be retrieved by `biomaRt`
    package.)

2.  Here is a [guideline](http://blog.genesino.com/2018/08/gene-tf/)
    about how to extract TF information from these two file(In Chinese).

Or maybe you prefer getting TF information from some prediction tools,
you can try [JASPAR](http://jaspar.genereg.net/) database.

# Some other analysis

## KEGG and GO enrichment analysis

``` r
go_bp=enrichGO( gene       = enzyme_info_refine$ensembl_gene_id,
                 OrgDb      = org.Mm.eg.db,
                 keyType    = 'ENSEMBL',
                 ont        = "BP", #'BP','CC','BF'
                 pAdjustMethod = "BH",
                 pvalueCutoff = 0.01,
                 qvalueCutoff = 0.05)

barplot(ego_bp,title="The GO_BP enrichment analysis of all DEGs")
```

![](Trans-omics-training_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

``` r
kk=enrichKEGG( gene      =enzyme_info_refine$entrezgene_id,
               organism = 'mmu',
               pvalueCutoff = 0.05)

barplot(kk, title="The KEGG enrichment analysis of all DEGs")
```

![](Trans-omics-training_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

## Get PTM information

``` r
library(iptmnetr)
#get protein PTM information, summarize dataframe by rbind()
res = get_substrates("Q9WVC3")
```

    ## No encoding supplied: defaulting to UTF-8.

``` r
View(res)
#get protein PTM-dependent PPI information
ptm_dependent_ppi <- get_ptm_dependent_ppi("Q15796")
View(ptm_dependent_ppi)
```
