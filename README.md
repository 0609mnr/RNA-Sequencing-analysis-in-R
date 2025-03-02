RNA-SEQUENCING ANALYSIS IN R
=============================
María Navarro-Riquelme

August 11, 2024



0.INTRODUCTION
---------------

RNA-Seq (RNA sequencing) is a high-throughput sequencing technique that allows for the comprehensive and large-scale analysis of the 
transcriptome, which is the complete set of RNA molecules present in a cell or tissue at a given time. Unlike traditional methods, 
RNA-Seq is not limited to detecting known genes; it enables the identification and quantification of gene expression, alternative 
transcripts, non-coding RNAs, and splicing events with high precision and sensitivity.


The RNA-Seq process generally begins with the extraction of total RNA or mRNA from the sample of interest. 
This RNA is then converted into complementary DNA (cDNA), which is fragmented and prepared for sequencing on 
a high-throughput sequencing platform. The resulting sequences are aligned to a reference genome or assembled 
de novo to quantify gene expression and other aspects of the transcriptome.


RNA-Seq is widely used in biomedical and biological research for various purposes, such as:

1. **Differential gene expression analysis**: Identifying genes whose expression significantly varies between different conditions or 
treatments.

2. **Comprehensive transcriptome characterization**: Discovering new transcripts and splicing variants that have not been previously 
annotated.

3. **Study of non-coding RNAs**: Investigating the function and regulation of non-coding RNAs, such as microRNAs and lncRNAs.

4. **Investigation of molecular mechanisms**: Gaining a better understanding of the underlying mechanisms in processes such as cell 
differentiation, stress response, or disease development.



Thanks to its ability to generate large amounts of detailed data and its flexibility in addressing diverse biological questions, 
RNA-Seq has become an essential tool in the field of molecular biology and bioinformatics. 

In this document, I provide a step-by-step RNA-Seq analysis using R tools, focusing on the processing, analysis, and visualization of the publicly available RNA-sequencing dataset from Parafati et al. 2023 (GSE234465). Here, I will only perform the analysis comparing two study groups that I have selected. 
 




1.ENVIRONMENT AND PACKAGES
---------------------------
When we are working with a large dataset, it is highly recommended to save the R environment to make the coding process faster and more effective.

```{r}
# Save R environment
save.image("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/GSE234465_Old_RNASeq.RData")

# Load R environment
load("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/GSE234465_Old_RNASeq.RData")
```


Load packages:

```{r, message=FALSE, warning=FALSE}
# Reading and viewing files
library(readxl)
library(openxlsx)
library(writexl)
library(tidyverse)
library(readr)
# Differential Gene Expression
library(edgeR)
library(DESeq2)
# Gene annotation 
library(AnnotationDbi)
library(org.Hs.eg.db)
# Data visualization
library(ggplot2)
library(EnhancedVolcano)
library(RColorBrewer)
library(knitr)
library(ggrepel)
# Gene set enrichment and pathway enrichment analysis and visualization
library(clusterProfiler)
library(pathview)
library(KEGGREST)
# Heatmap
library(ComplexHeatmap)
```

2.RESTRUCTURE THE DATA
-----------------------

When we carry out RNASeq analysis from fastq files, we'll obtain a separated txt file of raw counts for each sample we are working with. Probably, it will be compressed. Then, once txt.gz files for each sample are saved on the same folder, we'll be able to deal with them in R in order to create a unique data table of raw counts.

Normally, we'll obtain a txt file with two columns: one belonging to the Ensembl identifier and the other one to the number of counts.


3.IMPORT COUNT DATA
---------------------

```{r}
expression_data <- read_table("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/GSE234465_estimated-counts.txt")

expression_data <- as.data.frame(expression_data)
head(expression_data,5)
```

In case we are interested only in protein coding genes, we can filter dataset to ensure that we'll work just with the genes we are interested in. In this way, we can remove pseudogenes, lincRNA, antisense, snoRNA, snRNA, unknown genes, etc

```{r}
# Filter protein-coding genes using dplyr 
expression <- expression_data %>%
  filter(Biotype == "protein_coding")

head(expression,5)
```



4.MANIPULATING THE DATA
-------------------------
In this step, we'll establish the necessary format of the count table for further DEG analysis.

4.1.Establish **Ensembl ID** as row names. 
------------------------------------------

If we try to establish "Gene Symbol" as row names we'll encounter some troubles because duplication problems often occur.

```{r}
rownames(expression) <- expression[,1]
expression <- expression[,-c(1,2,3)]
```

Finally, the result is a DataFrame in which row names are Ensembl ID and column names are samples we are working with.


4.2.Transform DataFrame into matrix
------------------------------------
```{r}
counts_mtrx <- as.matrix(expression)
counts_mtrx <- apply(counts_mtrx, 2, as.numeric) # Counts must have numeric values
rownames(counts_mtrx) <- rownames(expression)
head(counts_mtrx,5)
```


4.3.Filtering the data
------------------------
Data can be filter out by mean reads per gene. This is a useful practice to reduce background and make dataset more manageable. In this case, we'll filter out genes that have at least a mean reads of 50. This value depends on the interest of each study.

```{r}
means <- rowMeans(counts_mtrx)
genes_mayor_50 <- rownames(counts_mtrx)[means > 50]
counts_mtrx <- counts_mtrx[genes_mayor_50, ]
counts_mtrx <- round(counts_mtrx,0)
head(counts_mtrx,5)
```




5.ASSOCIATE EACH SAMPLE TO A STUDY GROUP
------------------------------------------

5.1.Import sample information
------------------------------

This info can be built in an Excel file or in the same RStudio terminal.

```{r}
sample_info <- read_excel("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/GSE234465_sample_info.xlsx")

write.csv(sample_info, "GSE234465_sample_info.csv", row.names = FALSE)

colData <- read.csv("GSE234465_sample_info.csv", row.names = 1)
head(colData,5)
```


5.2.Establishment of groups
---------------------------

```{r}
DGEobj <- DGEList(counts_mtrx)
group <- colData$Group
DGEobj$samples$group <- group
DGEobj$sample
```



6.DIFFERENTIAL GENE EXPRESSION ANALYSIS
------------------------------------------

6.1.Create DGE object:
------------------------

```{r, message=FALSE, warning=FALSE}
DE_Group <- DESeqDataSetFromMatrix(countData = counts_mtrx, colData = colData, 
                                    design = ~ Group)
DE_Group <- DESeq(DE_Group)
```

IMPORTANT: countData must necessarily be raw counts matrix. We don't have to use a normalized counts table instead because DESeq2 does its own calculations for normalize the data.


6.2.Differential Gene Expression analysis between two groups: **Flight_Old vs. Ground_Old**
--------------------------------------------------------------------------------------------

```{r}
DE_Old <- results(DE_Group, contrast = c("Group","F_Old","G_Old"))

resDE_Old <- as.data.frame(DE_Old@listData) # Obtain DESeq2 results in a DataFrame format

rownames(resDE_Old) <- DE_Old@rownames

```

Number of significantly overexpressed genes:

```{r}
upreg_Old <- subset(resDE_Old, padj <= 0.05 & log2FoldChange > 1)
upreg_Old <- nrow(upreg_Old)
print(upreg_Old)
```



Number of significantly underexpressed genes:

```{r}
downreg_Old <- subset(resDE_Old, padj <= 0.05 & log2FoldChange < -1)
downreg_Old <- nrow(downreg_Old)
print(downreg_Old)
```



*Log Fold Change* is a measure used to compare gene expression in two different conditions or groups. It represents the magnitude of change in gene expression between these conditions.

-   LFC values close to 0 (LFC = 0) indicate no change in gene expression between the two conditions.
-   Positive LFC values (LFC \> 0) indicate that there is an increase in gene expression in condition A relative to B.
-   Negative LFC values (LFC \< 0) indicate that there is a decrease in gene expression in condition A relative to B.



6.3.Gene Annotation
---------------------
At this point, we can add some relevant info of genes such as Symbol, gene_id.

Assigning different gene information is often useful as it may be needed in subsequent analyses.

```{r, message=FALSE}
resDE_Old$Ensembl <- rownames(resDE_Old)

resDE_Old$Symbol <- mapIds(org.Hs.eg.db,
                             keys = row.names(resDE_Old),
                             column = "SYMBOL",
                             keytype = "ENSEMBL",
                             multiVals = "first")

resDE_Old$gene_id <- mapIds(org.Hs.eg.db,
                              keys = row.names(resDE_Old),
                              column = "ENTREZID",
                              keytype = "ENSEMBL",
                              multiVals = "first")


# OPTIONAL: Place the "Symbol" row as the first one to make the gene name more visible
resDE_Old <- resDE_Old[, c("Symbol", setdiff(names(resDE_Old), "Symbol"))]
head(resDE_Old,5)
```


6.4.Gene expression visualization using Volcano plot
-----------------------------------------------------
Once we've prepared DESeq2 results DataFrame, we can add a Group column based on the gene expression. 
We'll name genes with a padj \< 0.05 and log2FC \< -1 as down-regulated, while genes with a padj \< 0.05 and log2FC \> 1 as up-regulated.

```{r}
resDE_Old$Group <- ifelse(resDE_Old$padj < 0.05 & resDE_Old$log2FoldChange < -1,"Down-Regulated", 
                            ifelse(resDE_Old$padj < 0.05 & resDE_Old$log2FoldChange > 1,"Up-Regulated", 
                                   "Not Significant"))
head(resDE_Old,5)
```


Generate a new DataFrame that contains just genes that meet the criteria of significance and change in expression

```{r fig.width=5, fig.height=3}
sig_Old <- subset(resDE_Old, padj < 0.05 & log2FoldChange > 1 | padj < 0.05 & log2FoldChange < -1)
```


Draw the Volcano plot

```{r warning=FALSE, fig.width=10, fig.height=8}
ggplot(resDE_Old, aes(x = log2FoldChange, 
                      y = -log10(padj), 
                      color = resDE_Old$Group)) + # assign axis and color genes based on 
                                                  # "Group" column (significance)
  geom_point(cex = 1) +   # shaping the points
  geom_text_repel(data = sig_Old, 
                  vjust = -1.5, 
                  size = 3, 
                  colour = "black", 
                  label = sig_Old$Symbol, 
                  box.padding = unit(0.01, "lines"), # label just the significant genes
                  hjust = 1,
                  max.overlaps = 10) +
  labs(title = "DEG Flight_Old vs Ground_Old",
       x = "Log2FC",
       y = "-log10(padj)") +
  geom_vline(xintercept = c(-1,1), # Log2FC threshold
             colour = "#666666", 
             linetype = "dashed") + # draw x line to make remarkable significant log2FC point
  geom_hline(yintercept = 1.30103,  # -log10(padj)= -log10(0.05)=1.30103
             colour = "#666666", 
             linetype = "dashed") + # draw y line to make remarkable significant padj 
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", 
                               "Not Significant" = "#CCCCCC", 
                               "Down-Regulated" = "#000099")) +  # color expression groups
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", 
                                "Not Significant" = "#CCCCCC", 
                                "Down-Regulated" = "#000099")) +  # color expression groups
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 10),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank()) +
  guides(color = guide_legend(title = NULL)) + # remove legend title
  theme_minimal()

```



7.SAMPLE CLUSTERING
--------------------

HEATMAP
-------

```{r, fig.width=8, fig.height=10}
# Filtering out DESeq2 results by padj significance
sigs_Old <- resDE_Old[resDE_Old$padj < 0.05,]

# Additional filter by baseMean and log2FC
sigs_Old <- sigs_Old[(sigs_Old$baseMean > 100) & (abs(sigs_Old$log2FoldChange) > 2),]

# Counts normalization
mat <- counts(DE_Group, normalized = T)[rownames(sigs_Old),]

# Standardization of data by row (gene)
mat.z <- t(apply(mat, 1, scale))

# Assigning column names to standarized matrix
colnames(mat.z) <- rownames(colData)

# Creating heatmap
Heatmap(mat.z, cluster_rows = T, cluster_columns = T, column_labels = colnames(mat.z),
        name = "Z-score", row_labels = sigs_Old[rownames(mat.z),]$Symbol)

```

PCA PLOT
--------

```{r, fig.width=6, fig.height=4}

# Variance stabilizing transformation:
vsdata <- vst(DE_Group, blind = FALSE)

# Transformed values to generate a PCA plot:
pca_data <- plotPCA(vsdata, intgroup = "Group", returnData = TRUE)

percent_variance <- round(100 * attr(pca_data, "percentVar"), 1)

# Assigning different color to each group of study
colors <- c("F_Old" = "#3300FF", 
            "F_Young" = "#009966", 
            "G_Old" = "#9966CC", 
            "G_Young" = "#CCFF99") 

# Creating PCA plot
ggplot(pca_data, aes(PC1, PC2, color=Group, label=rownames(pca_data))) +
  geom_point(aes(color=Group), size=3) +
  scale_color_manual(values = colors) +
  xlab(paste0("PC1: ",percent_variance[1],"% variance")) +
  ylab(paste0("PC2: ",percent_variance[2],"% variance")) +
  ggtitle("PCA plot") +
  theme_minimal() +
  theme(legend.title = element_blank()) +
  guides(color = guide_legend(override.aes = list(label = levels(pca_data$Group))))

```




8.ENRICHMENT OF GENE SETS AND PATHWAYS
---------------------------------------

In order to clarify what gene sets and pathways are up-regulated and down-regulated we'll extract a subset for each group based on the padj and log2FC value, being log2FC \> 1 as threshold to up-regulated genes and log2FC \< -1 as threshold to down-regulated genes. Padj threshold is \< 0.05 for both.

To carry out Gene Ontology and KEGG pathways analysis, gene_id is needed as reference.

```{r}
# We need to create geneList variable for Gene Ontology analysis. 
# The variable has to contain the list of gene_ids that have been used in the 
# differential gene expression.
geneList_Old <- resDE_Old[,9] 
head(geneList_Old,5)
```


8.1.ENRICHMENT OF UP-REGULATED GENES
-------------------------------------

```{r}
UP_Old <- subset(resDE_Old, padj <= 0.05 & log2FoldChange > 1)

de_UP_Old <- UP_Old$gene_id[UP_Old$padj < 0.05]

head(de_UP_Old,5)
```


8.1.1.GENE ONTOLOGY:
--------------------

Biological Processes (BP)
------------------------

```{r, fig.width=10, fig.height=5}
enrich_go_BP_Old_UP <- enrichGO(gene = de_UP_Old, # List of significantly up-regulated genes
                                universe = geneList_Old,  # List of genes used in DGE analysis
                                keyType = "ENTREZID",
                                pvalueCutoff = 0.05,
                                pAdjustMethod = "BH",
                                qvalueCutoff = 0.05,
                                OrgDb = org.Hs.eg.db,
                                ont = "BP") # Specify Gene Ontology: BP,MF,CC,ALL

# Obtaining the DataFrame from the data
enrich_go_BP_Old_UP <- enrich_go_BP_Old_UP@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_go_BP_Old_UP <- subset(enrich_go_BP_Old_UP, -log10(enrich_go_BP_Old_UP$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(enrich_go_BP_Old_UP, aes(x = reorder(Description, 
                                            -log10(p.adjust), 
                                            decreasing = FALSE), 
                                y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "coral1") +
  geom_text(aes(label = Count), size = 3, hjust = -0.5) +  
  labs(title = "Up-regulated Biological Processes in Flight Old", 
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(size = 9),
        axis.text.y = element_text(size = 10)) +
  coord_flip()
```


Molecular Functions (MF)
------------------------

```{r, fig.width=10, fig.height=5}
enrich_go_MF_Old_UP <- enrichGO(gene = de_UP_Old,  # List of significantly up-regulated genes
                                universe = geneList_Old,  # List of genes used in DGE analysis
                                keyType = "ENTREZID",
                                pvalueCutoff = 0.05,
                                pAdjustMethod = "BH",
                                qvalueCutoff = 0.05,
                                OrgDb = org.Hs.eg.db,
                                ont = "MF") # Specify Gene Ontology: BP,MF,CC,ALL

# Obtaining the DataFrame from the data
enrich_go_MF_Old_UP <- enrich_go_MF_Old_UP@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_go_MF_Old_UP <- subset(enrich_go_MF_Old_UP, -log10(enrich_go_MF_Old_UP$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(enrich_go_MF_Old_UP, aes(x = reorder(Description, 
                                            -log10(p.adjust), 
                                            decreasing = FALSE), 
                                y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "coral2", width = 0.2) +
  geom_text(aes(label = Count), size = 3, hjust = -0.5) +  
  labs(title = "Up-regulated Molecular Functions in Flight Old", 
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(size = 9),
        axis.text.y = element_text(size = 10)) +
  coord_flip()

```

-\> Not enriched Molecular Functions.


Cellular Components (CC)
-------------------------

```{r, fig.width=10, fig.height=5}
enrich_go_CC_Old_UP <- enrichGO(gene = de_UP_Old,  # List of significantly up-regulated genes
                                universe = geneList_Old,  # List of genes used in DGE analysis
                                keyType = "ENTREZID",
                                pvalueCutoff = 0.05,
                                pAdjustMethod = "BH",
                                qvalueCutoff = 0.05,
                                OrgDb = org.Hs.eg.db,
                                ont = "CC") # Specify Gene Ontology: BP,MF,CC,ALL

# Obtaining the DataFrame from the data
enrich_go_CC_Old_UP <- enrich_go_CC_Old_UP@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_go_CC_Old_UP <- subset(enrich_go_CC_Old_UP, -log10(enrich_go_CC_Old_UP$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(enrich_go_CC_Old_UP, aes(x = reorder(Description, 
                                            -log10(p.adjust), 
                                            decreasing = FALSE), 
                                y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "coral3", width = 0.3) +
  geom_text(aes(label = Count), size = 3, hjust = -0.5) +  
  labs(title = "Up-regulated Cellular Components in Flight Old",
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(size = 9),
        axis.text.y = element_text(size = 10)) +
  coord_flip()

```

-\> Not enriched Cellular Components.





8.1.2.PATHWAY ENRICHMENT (KEGG - Kyoto Encyclopedia of Genes and Genomes):
---------------------------------------------------------------------------

```{r, fig.width=10, fig.height=8}
enrich_kegg_Old_UP <- enrichKEGG(gene = de_UP_Old,  # List of significantly up-regulated genes
                                 organism = "hsa",
                                 keyType = "kegg",
                                 pAdjustMethod = "BH",
                                 pvalueCutoff = 0.05,
                                 qvalueCutoff = 0.05)

# Obtaining the DataFrame from the data
enrich_kegg_Old_UP <- enrich_kegg_Old_UP@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_kegg_Old_UP <- subset(enrich_kegg_Old_UP, -log10(enrich_kegg_Old_UP$p.adjust) > 1.3)

# Sort the pathway descriptions according to the p.adjust:
enrich_kegg_Old_UP$Description <- factor(enrich_kegg_Old_UP$Description,
                                        levels = enrich_kegg_Old_UP$Description
                                        [order(enrich_kegg_Old_UP$p.adjust)])

# Barplot visualization of results sorted by their significance
ggplot(enrich_kegg_Old_UP, aes(x = Description, 
                               y = Count, 
                               fill = p.adjust)) +
  geom_bar(stat = "identity") +
  scale_fill_gradient(low = "blue", high = "red") +  # Gradient color scale based on p.adjust
  labs(x = "", 
       y = "Counts", 
       title = "UP-regulated KEGG pathways in Flight Old") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1, size = 10),
        plot.margin = unit(c(1, 1, 1, 5), "lines"), 
        panel.background = element_rect(fill = "lightgray"))

```



8.1.3.GENE SET ENRICHMENT - GSEA (KEGG - Kyoto Encyclopedia of Genes and Genomes):
---------------------------------------------------------------------------------

In this case, the input must be a table with the gene_id and log2FC

```{r, warning=FALSE, fig.width=6, fig.height=4}
gene_id_log2FC_Old_UP <- UP_Old$log2FoldChange 

names(gene_id_log2FC_Old_UP) <- UP_Old$gene_id

# Omit NA values 
gene_id_log2FC_Old_UP <- na.omit(gene_id_log2FC_Old_UP)

# Sort in descending order (supposedly required for clusterProfiler)
sort_gene_id_log2FC_Old_UP <- sort(gene_id_log2FC_Old_UP, decreasing = TRUE)

# Enrichment analysis
gse_kegg_Old_UP <- gseKEGG(geneList = sort_gene_id_log2FC_Old_UP,
                           pvalueCutoff = 0.05,
                           verbose = TRUE,
                           organism = "hsa",
                           pAdjustMethod = "BH")

# Obtaining the DataFrame from the data
gse_kegg_Old_UP <- gse_kegg_Old_UP@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
gse_kegg_Old_UP <- subset(gse_kegg_Old_UP, -log10(gse_kegg_Old_UP$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(gse_kegg_Old_UP, aes(x = reorder(gse_kegg_Old_UP$Description, 
                                        -log10(p.adjust), 
                                        decreasing = TRUE), 
                            y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "#CC3366") +
  labs(title = "UP-Regulated KEGG Gene Set Enrichment in Flight Old", 
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 10),
        plot.margin = unit(c(1, 1, 1, 3), "lines"))

```

-\> Not enriched Gene Sets.



8.2.ENRICHMENT OF DOWN-REGULATED GENES
--------------------------------------

```{r}
DOWN_Old <- subset(resDE_Old, padj <= 0.05 & log2FoldChange < -1)

de_DOWN_Old <- DOWN_Old$gene_id[DOWN_Old$padj < 0.05]

head(de_DOWN_Old,5)
```



8.2.1.GENE ONTOLOGY:
--------------------
Biological Processes (BP)
-------------------------

```{r, fig.width=10, fig.height=5}
enrich_go_BP_Old_DOWN <- enrichGO(de_DOWN_Old, # List of significantly down-regulated genes
                                  universe = geneList_Old, # List of genes used in DGE analysis
                                  keyType = "ENTREZID",
                                  pvalueCutoff = 0.05,
                                  pAdjustMethod = "BH",
                                  qvalueCutoff = 0.05,
                                  OrgDb = org.Hs.eg.db,
                                  ont = "BP") # Specify Gene Ontology: BP,MF,CC,ALL

# Obtaining the DataFrame from the data
enrich_go_BP_Old_DOWN <- enrich_go_BP_Old_DOWN@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_go_BP_Old_DOWN <- subset(enrich_go_BP_Old_DOWN, -log10(enrich_go_BP_Old_DOWN$p.adjust) > 2)  
# We can adjust p.adjusted value to achieve a better visualization of data 

# Barplot visualization of results 
ggplot(enrich_go_BP_Old_DOWN, aes(x = reorder(Description, 
                                              -log10(p.adjust), 
                                              decreasing = FALSE), 
                                  y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "cadetblue2") +
  geom_text(aes(label = Count), size = 3, hjust = -0.5) +  
  labs(title = "Down-regulated Biological Processes in Flight Old", 
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(size = 9),
        axis.text.y = element_text(size = 10)) +
  coord_flip() 

```


Molecular Function (MF)
-----------------------
```{r, fig.width=12, fig.height=5}
enrich_go_MF_Old_DOWN <- enrichGO(de_DOWN_Old,  # List of significantly down-regulated genes
                                  universe = geneList_Old,  # List of genes used in DGE analysis
                                  keyType = "ENTREZID",
                                  pvalueCutoff = 0.05,
                                  pAdjustMethod = "BH",
                                  qvalueCutoff = 0.05,
                                  OrgDb = org.Hs.eg.db,
                                  ont = "MF") # Specify Gene Ontology: BP,MF,CC,ALL

# Obtaining the DataFrame from the data
enrich_go_MF_Old_DOWN <- enrich_go_MF_Old_DOWN@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_go_MF_Old_DOWN <- subset(enrich_go_MF_Old_DOWN, -log10(enrich_go_MF_Old_DOWN$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(enrich_go_MF_Old_DOWN, aes(x = reorder(Description, 
                                              -log10(p.adjust), 
                                              decreasing = FALSE), 
                                  y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "cadetblue3", width = 0.5) +
  geom_text(aes(label = Count), size = 3, hjust = -0.5) +  
  labs(title = "Down-regulated Molecular Functions in Flight Old", 
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(size = 9),
        axis.text.y = element_text(size = 9)) +
  coord_flip() 
```



Cellular Component (CC)
-------------------------

```{r, fig.width=10, fig.height=5}
enrich_go_CC_Old_DOWN <- enrichGO(de_DOWN_Old,  # List of significantly down-regulated genes
                                  universe = geneList_Old,  # List of genes used in DGE analysis
                                  keyType = "ENTREZID",
                                  pvalueCutoff = 0.05,
                                  pAdjustMethod = "BH",
                                  qvalueCutoff = 0.05,
                                  OrgDb = org.Hs.eg.db,
                                  ont = "CC") # Specify Gene Ontology: BP,MF,CC,ALL

# Obtaining the DataFrame from the data
enrich_go_CC_Old_DOWN <- enrich_go_CC_Old_DOWN@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_go_CC_Old_DOWN <- subset(enrich_go_CC_Old_DOWN, -log10(enrich_go_CC_Old_DOWN$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(enrich_go_CC_Old_DOWN, aes(x = reorder(Description, 
                                              -log10(p.adjust), 
                                              decreasing = FALSE), 
                                  y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "cadetblue", width = 0.5) +
  geom_text(aes(label = Count), size = 3, hjust = -0.5) +  
  labs(title = "Down-regulated Cellular Components in Flight Old",
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(size = 9),
        axis.text.y = element_text(size = 10)) +
  coord_flip() 
```





8.2.2.PATHWAY ENRICHMENT (KEGG - Kyoto Encyclopedia of Genes and Genomes):
--------------------------------------------------------------------------

```{r, fig.width=10, fig.height=5}
enrich_kegg_Old_DOWN <- enrichKEGG(gene = de_DOWN_Old,  # List of significantly down-regulated genes
                                   organism = "hsa",
                                   keyType = "kegg",
                                   pAdjustMethod = "BH",
                                   pvalueCutoff = 0.05,
                                   qvalueCutoff = 0.05)

# Obtaining the DataFrame from the data
enrich_kegg_Old_DOWN <- enrich_kegg_Old_DOWN@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
enrich_kegg_Old_DOWN <- subset(enrich_kegg_Old_DOWN, -log10(enrich_kegg_Old_DOWN$p.adjust) > 1.3)

# Sort the pathway descriptions according to the p.adjust:
enrich_kegg_Old_DOWN$Description <- factor(enrich_kegg_Old_DOWN$Description,
                                           levels = enrich_kegg_Old_DOWN$Description
                                           [order(enrich_kegg_Old_DOWN$p.adjust)])

# Barplot visualization of results sorted by their significance
ggplot(enrich_kegg_Old_DOWN, aes(x = Description, 
                                 y = Count,
                                 fill = p.adjust)) +
  geom_bar(stat = "identity") +
  scale_fill_gradient(low = "blue", high = "red") +  # Gradient color scale based on p.adjust
  labs(x = "", 
       y = "Counts", 
       title = "Down-Regulated KEGG pathways in Flight Old") +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 10),
        plot.margin = unit(c(1, 1, 1, 3), "lines"))
```





8.2.3.GENE SET ENRICHMENT - GSEA (KEGG - Kyoto Encyclopedia of Genes and Genomes):
----------------------------------------------------------------------------------

In this case, the input must be a table with the gene_id and log2FC

```{r, warning=FALSE, fig.width=6, fig.height=4}
gene_id_log2FC_Old_DOWN <- DOWN_Old$log2FoldChange 

names(gene_id_log2FC_Old_DOWN) <- DOWN_Old$gene_id

# Omit NA values 
gene_id_log2FC_Old_DOWN <- na.omit(gene_id_log2FC_Old_DOWN)

# Sort in descending order (supposedly required for clusterProfiler)
sort_gene_id_log2FC_Old_DOWN <- sort(gene_id_log2FC_Old_DOWN, decreasing = TRUE)

# Enrichment analysis
gse_kegg_Old_DOWN <- gseKEGG(geneList = sort_gene_id_log2FC_Old_DOWN,
                             pvalueCutoff = 0.05,
                             verbose = TRUE,
                             organism = "hsa",
                             pAdjustMethod = "BH")

# Obtaining the DataFrame from the data
gse_kegg_Old_DOWN <- gse_kegg_Old_DOWN@result

# Filtering by padj <= 0.05 (-log10(padj=0.05) = 1.30103)
gse_kegg_Old_DOWN <- subset(gse_kegg_Old_DOWN, -log10(gse_kegg_Old_DOWN$p.adjust) > 1.3)

# Barplot visualization of results 
ggplot(gse_kegg_Old_DOWN, aes(x = reorder(gse_kegg_Old_DOWN$Description, 
                                          -log10(p.adjust), 
                                          decreasing = TRUE), 
                              y = -log10(p.adjust))) +
  geom_bar(stat = "identity", fill = "#000099") +
  labs(title = "Down-Regulated KEGG Gene Set Enrichment in Flight Old", 
       x = NULL, 
       y = "-log10(padj)") +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 10),
        plot.margin = unit(c(1, 1, 1, 3), "lines"))
```

-\> Not enriched Gene Sets




8.3.PATHWAYS VISUALIZATION WITH PATHVIEW
----------------------------------------

Once the enrichment analysis is done, we can visualize the expression of the genes within a KEGG pathway using *pathview* library.

In this way, we can obtain an image of a given pathway in which down-regulated genes are colored green and up-regulated genes are colored red.

```{r}
# Selection of significantly differentiated genes (up-regulated and down-regulated)
significant_genes_Old <- subset(resDE_Old, padj <= 0.05 & (log2FoldChange > 1 | log2FoldChange < -1))

# Create a table with gene_id and log2FC. This variable will be gene.data argument
genes_pathview <- significant_genes_Old$log2FoldChange
names(genes_pathview) <- significant_genes_Old$gene_id

head(genes_pathview,5)

```

NOTE: It's highly recommended to create this variable from our DEG analysis in order to use the same list of genes. This makes the analysis more precise and specific.



```{r pathview-figure, echo=TRUE, message=FALSE}
# AMPK Signaling Pathway
pv_AMPK <- pathview(gene.data = genes_pathview,  
                    pathway.id = "hsa04152",  # Here we have to specify the ID to visualize certain pathway
                    species = "hsa",
                    gene.idtype = "entrez",
                    kegg.native = TRUE,
                    out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/AMPK_Sign_Path_hsa04152_pathview.png")

# Fructose and mannose metabolism:
pv_Fruct_mannose <- pathview(gene.data = genes_pathview,  
                             pathway.id = "hsa00051",  # Here we have to specify the ID to  visualize certain pathway
                             species = "hsa",
                             gene.idtype = "entrez",
                             kegg.native = TRUE,
                             out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Fruct_mannose_hsa00051_pathview.png")

# PPAR Signaling Pathway
pv_PPAR <- pathview(gene.data = genes_pathview,  
                    pathway.id = "hsa03320",  # Here we have to specify the ID to visualize certain pathway
                    species = "hsa",
                    gene.idtype = "entrez",
                    kegg.native = TRUE,
                    out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/PPAR_Sign_Path_hsa03320.pathview.png")

# Ribosome
pv_Ribosome <- pathview(gene.data = genes_pathview,  
                    pathway.id = "hsa03010",  # Here we have to specify the ID to visualize certain pathway
                    species = "hsa",
                    gene.idtype = "entrez",
                    kegg.native = TRUE,
                    out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Ribosome_hsa03010.pathview.png")

# Glucagon Signaling Pathway
pv_Glucagon <- pathview(gene.data = genes_pathview,  
                    pathway.id = "hsa04922",  # Here we have to specify the ID to visualize certain pathway
                    species = "hsa",
                    gene.idtype = "entrez",
                    kegg.native = TRUE,
                    out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Glucagon_Sign_Path_hsa04922.pathview.png")

# Cytoskeleton in muscle cells
pv_Cytoskeleton_muscle_cells <- pathview(gene.data = genes_pathview,  
                                         pathway.id = "hsa04820",  # Here we have to specify the ID to visualize certain pathway
                                         species = "hsa",
                                         gene.idtype = "entrez",
                                         kegg.native = TRUE,
                                         out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Cytoskeleton_muscle_cells_hsa04820.pathview.png")

# ECM-receptor interaction
pv_ECM <- pathview(gene.data = genes_pathview,  
                   pathway.id = "hsa04512",  # Here we have to specify the ID to visualize certain pathway
                   species = "hsa",
                   gene.idtype = "entrez",
                   kegg.native = TRUE,
                   out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/ECM_recept_int_hsa04512_pathview.png")

# Focal adhesion
pv_Foc_adh <- pathview(gene.data = genes_pathview,  
                       pathway.id = "hsa04510",  # Here we have to specify the ID to visualize certain pathway
                       species = "hsa",
                       gene.idtype = "entrez",
                       kegg.native = TRUE,
                       out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Focal_adhesion_hsa04510_pathview.png")

# Regulation of actin cytoskeleton
pv_Regul_actin_cytoskeleton <- pathview(gene.data = genes_pathview,  
                                        pathway.id = "hsa04810",  # Here we have to specify the ID to visualize certain pathway
                                        species = "hsa",
                                        gene.idtype = "entrez",
                                        kegg.native = TRUE,
                                        out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Regulation_actin_cytoskeleton_hsa04810.pathview.png")

# Hypertrophic cardiomyopathy
pv_Hypertrophic_cardiomyopathy <- pathview(gene.data = genes_pathview,  
                                        pathway.id = "hsa05410",  # Here we have to specify the ID to visualize certain pathway
                                        species = "hsa",
                                        gene.idtype = "entrez",
                                        kegg.native = TRUE,
                                        out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Hypertrophic_cardiomyopathy_hsa05410.pathview.png")

# Dilated cardiomyopathy
pv_Dilated_cardiomyopathy <- pathview(gene.data = genes_pathview,  
                                           pathway.id = "hsa05414",  # Here we have to specify the ID to visualize certain pathway
                                           species = "hsa",
                                           gene.idtype = "entrez",
                                           kegg.native = TRUE,
                                           out.suffix = "pathview")

include_graphics("C:/Users/MARIA/Desktop/BIOINFORMATIC/PRUEBA RNASeq/Figures/Dilated_cardiomyopathy_hsa05414.pathview.png")

```







8.4.VISUALIZATION OF GENE EXPRESSION OF A PATHWAY WITH KEGGREST
---------------------------------------------------------------

Here, we can obtain a dot plot representing the expression of genes within a given pathway.

```{r, message=FALSE}
# Obtaining KEGG links for human pathways and genes:
hsa_path_eg <- keggLink("pathway", "hsa") %>%
  tibble(pathway = ., eg = sub("hsa:", "", names(.)))

# Add symbol annotations and Ensembl for genes:
hsa_kegg_anno <- hsa_path_eg %>%
  mutate(
    symbol = mapIds(org.Hs.eg.db, eg, "SYMBOL", "ENTREZID"),
    ensembl = mapIds(org.Hs.eg.db, eg, "ENSEMBL", "ENTREZID")
  )

```



Biosynthesis of unsaturated fatty acids (hsa01040):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa01040_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa01040", ]

hsa01040_path <- as.data.frame(hsa01040_path)

colnames(hsa01040_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa01040_merge_Old <- merge(hsa01040_path, resDE_Old, by = "Symbol")
head(hsa01040_merge_Old,5)

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa01040_Old_expr_genes <- hsa01040_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa01040_Old_expr_genes$Group <- ifelse(hsa01040_Old_expr_genes$padj < 0.05 & hsa01040_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa01040_Old_expr_genes$padj < 0.05 & hsa01040_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))
head(hsa01040_Old_expr_genes,5)

# 5.Visualization of results
ggplot(hsa01040_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Biosynthesis of unsaturated fatty acids (hsa01040)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 8),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())

```



AMPK Signaling Pathway (hsa04152):
```{r, fig.width=12, fig.height=6}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa04152_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa04152", ]

hsa04152_path <- as.data.frame(hsa04152_path)

colnames(hsa04152_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa04152_merge_Old <- merge(hsa04152_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa04152_Old_expr_genes <- hsa04152_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa04152_Old_expr_genes$Group <- ifelse(hsa04152_Old_expr_genes$padj < 0.05 & hsa04152_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa04152_Old_expr_genes$padj < 0.05 & hsa04152_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa04152_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: AMPK Signaling Pathway (hsa04152)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())

```



Biosynthesis of amino acids (hsa01230):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa01230_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa01230", ]

hsa01230_path <- as.data.frame(hsa01230_path)

colnames(hsa01230_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa01230_merge_Old <- merge(hsa01230_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa01230_Old_expr_genes <- hsa01230_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa01230_Old_expr_genes$Group <- ifelse(hsa01230_Old_expr_genes$padj < 0.05 & hsa01230_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa01230_Old_expr_genes$padj < 0.05 & hsa01230_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa01230_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Biosynthesis of amino acids (hsa01230)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Fructose and mannose metabolism (hsa00051):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa00051_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa00051", ]

hsa00051_path <- as.data.frame(hsa00051_path)

colnames(hsa00051_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa00051_merge_Old <- merge(hsa00051_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa00051_Old_expr_genes <- hsa00051_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa00051_Old_expr_genes$Group <- ifelse(hsa00051_Old_expr_genes$padj < 0.05 & hsa00051_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa00051_Old_expr_genes$padj < 0.05 & hsa00051_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa00051_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Fructose and mannose metabolism (hsa00051)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



PPAR signaling pathway (hsa03320):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa03320_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa03320", ]

hsa03320_path <- as.data.frame(hsa03320_path)

colnames(hsa03320_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa03320_merge_Old <- merge(hsa03320_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa03320_Old_expr_genes <- hsa03320_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa03320_Old_expr_genes$Group <- ifelse(hsa03320_Old_expr_genes$padj < 0.05 & hsa03320_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa03320_Old_expr_genes$padj < 0.05 & hsa03320_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa03320_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: PPAR signaling pathway (hsa03320)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Glucagon signaling pathway (hsa04922):
```{r, fig.width=12, fig.height=6}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa04922_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa04922", ]

hsa04922_path <- as.data.frame(hsa04922_path)

colnames(hsa04922_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa04922_merge_Old <- merge(hsa04922_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa04922_Old_expr_genes <- hsa04922_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa04922_Old_expr_genes$Group <- ifelse(hsa04922_Old_expr_genes$padj < 0.05 & hsa04922_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa04922_Old_expr_genes$padj < 0.05 & hsa04922_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa04922_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Glucagon signaling pathway (hsa04922)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Fatty acid metabolism (hsa01212):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa01212_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa01212", ]

hsa01212_path <- as.data.frame(hsa01212_path)

colnames(hsa01212_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa01212_merge_Old <- merge(hsa01212_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa01212_Old_expr_genes <- hsa01212_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa01212_Old_expr_genes$Group <- ifelse(hsa01212_Old_expr_genes$padj < 0.05 & hsa01212_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa01212_Old_expr_genes$padj < 0.05 & hsa01212_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa01212_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Fatty acid metabolism (hsa01212)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Cytoskeleton in muscle cells (hsa04820):
```{r, fig.width=12, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa04820_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa04820", ]

hsa04820_path <- as.data.frame(hsa04820_path)

colnames(hsa04820_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa04820_merge_Old <- merge(hsa04820_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa04820_Old_expr_genes <- hsa04820_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa04820_Old_expr_genes$Group <- ifelse(hsa04820_Old_expr_genes$padj < 0.05 & hsa04820_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa04820_Old_expr_genes$padj < 0.05 & hsa04820_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa04820_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Cytoskeleton in muscle cells (hsa04820)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 8),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



ECM-receptor interaction (hsa04512):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa04512_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa04512", ]

hsa04512_path <- as.data.frame(hsa04512_path)

colnames(hsa04512_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa04512_merge_Old <- merge(hsa04512_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa04512_Old_expr_genes <- hsa04512_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa04512_Old_expr_genes$Group <- ifelse(hsa04512_Old_expr_genes$padj < 0.05 & hsa04512_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa04512_Old_expr_genes$padj < 0.05 & hsa04512_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa04512_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: ECM-receptor interaction (hsa04512)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Focal adhesion (hsa04510):
```{r, fig.width=12, fig.height=6}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa04510_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa04510", ]

hsa04510_path <- as.data.frame(hsa04510_path)

colnames(hsa04510_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa04510_merge_Old <- merge(hsa04510_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa04510_Old_expr_genes <- hsa04510_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa04510_Old_expr_genes$Group <- ifelse(hsa04510_Old_expr_genes$padj < 0.05 & hsa04510_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa04510_Old_expr_genes$padj < 0.05 & hsa04510_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa04510_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Focal adhesion (hsa04510)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Regulation of actin cytoskeleton (hsa04810):
```{r, fig.width=12, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa04810_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa04810", ]

hsa04810_path <- as.data.frame(hsa04810_path)

colnames(hsa04810_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa04810_merge_Old <- merge(hsa04810_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa04810_Old_expr_genes <- hsa04810_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa04810_Old_expr_genes$Group <- ifelse(hsa04810_Old_expr_genes$padj < 0.05 & hsa04810_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa04810_Old_expr_genes$padj < 0.05 & hsa04810_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa04810_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Regulation of actin cytoskeleton (hsa04810)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 8),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Hypertrophic cardiomyopathy (hsa05410):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa05410_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa05410", ]

hsa05410_path <- as.data.frame(hsa05410_path)

colnames(hsa05410_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa05410_merge_Old <- merge(hsa05410_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hsa05410_Old_expr_genes <- hsa05410_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hsa05410_Old_expr_genes$Group <- ifelse(hsa05410_Old_expr_genes$padj < 0.05 & hsa05410_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hsa05410_Old_expr_genes$padj < 0.05 & hsa05410_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hsa05410_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Hypertrophic cardiomyopathy (hsa05410)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```



Dilated cardiomyopathy (hsa05414):
```{r, fig.width=10, fig.height=5}
# 1.Filtering of KEGG annotations for a specific pathway:
hsa05414_path <- hsa_kegg_anno[hsa_kegg_anno$pathway == "path:hsa05414", ]

hsa05414_path <- as.data.frame(hsa05414_path)

colnames(hsa05414_path)[3] <- "Symbol"

# 2.Combination of KEGG annotation data with differential expression results:
hsa05414_merge_Old <- merge(hsa05414_path, resDE_Old, by = "Symbol")

# 3.Selection of relevant columns for analysis (Symbol, log2FC, p-value, padj, gene_id)
hhsa05414_Old_expr_genes <- hsa05414_merge_Old[, c(1,6,9,10,12)]

# 4.Classification of genes according to their differential expression:
hhsa05414_Old_expr_genes$Group <- ifelse(hhsa05414_Old_expr_genes$padj < 0.05 & hhsa05414_Old_expr_genes$log2FoldChange < -1,"Down-Regulated", 
                                        ifelse(hhsa05414_Old_expr_genes$padj < 0.05 & hhsa05414_Old_expr_genes$log2FoldChange > 1,"Up-Regulated",
                                               "Not Significant"))

# 5.Visualization of results
ggplot(hhsa05414_Old_expr_genes, aes(x = Symbol, y = log2FoldChange, fill = Group, color = Group)) + 
  geom_point(cex = 2) +  
  labs(title = expression(bold("Gene Expression: Dilated cardiomyopathy (hsa05414)")),
       x = "Genes",
       y = "log2FC") +
  geom_hline(yintercept = 0, colour = "grey", linetype = "dashed") +
  scale_fill_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  scale_color_manual(values = c("Up-Regulated" = "#CC3366", "Not Significant" = "#CCCCCC", "Down-Regulated" = "#000099")) +
  theme(axis.text.x = element_text(angle = 60, hjust = 1, size = 9),
        plot.margin = unit(c(1, 1, 1, 1), "lines"),
        legend.title = element_blank())
```




9.ADDITIONAL INFO
-----------------

Some useful databases:

-   GeneCards to search info about certain gene: <https://www.genecards.org/>

-   KEGG - Kyoto Encyclopedia of Genes and Genomes to search info about pathways and others: <https://www.genome.jp/kegg/>

-   Gene Expression Omnibus (GEO) to select public dataset: <https://www.ncbi.nlm.nih.gov/geo/>

-   The Human Protein Atlas to search info about certain protein: <https://www.proteinatlas.org/>

-   IntAct Molecular Interaction Database to analyze molecular interactions between proteins: <https://www.ebi.ac.uk/intact/home>
