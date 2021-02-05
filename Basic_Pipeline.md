Basic Pipeline for scRNAseq Data Analysis
================
Instructors : Somi Kim, Eunseo Park, Donggon Cha
2021/02/20

# Introduction

# Quality Control (Cell / Gene QC)

## Empty droplet filtering

## Removal of low quality cells

To preprocess liver cancer scRNA-seq data, basic information of each cell of experimental design is necessary. Given sample information is loaded and organized. First, we set sample ID per each patient from its sample name to include liver cancer subtype information. We also set Diagnosis as liver cancer subtypes which are HCC or iCCA in this data.

``` r
library(DropletUtils)
library(dplyr)
library(scater)

dir = "E:/Project/KOGO_2021/rawdata"

sampleInfo <- read.table(paste0(dir, "/samples.txt"), header=TRUE, sep="\t")
head(sampleInfo)

sampleInfo$ID <- sapply(sampleInfo$Sample %>% as.character(), function(x) {strsplit(x, split="_")[[1]][3]}) 
sampleInfo$ID <- gsub("LCP", "", sampleInfo$ID)

orig.ids = sampleInfo$ID %>% as.factor() %>% levels()
new.ids = c("H18", "H21", "H23", "C25", "C26", "H28", 
            "C29", "H30", "C35", "H37", "H38", "C39")

for(i in orig.ids){
  new.id = new.ids[grep(i, new.ids)]
  sampleInfo[grep(i, sampleInfo$ID),]$ID = new.id
}
sampleInfo$Diagnosis = sampleInfo$ID
for(i in c("H", "C")){
  if(i == "H"){
    sampleInfo[grep(i, sampleInfo$ID),]$Diagnosis = "HCC"
  }else{
    sampleInfo[grep(i, sampleInfo$ID),]$Diagnosis = "iCCA"
  }
}
```

For preprocessing of scRNA-seq data, mapped reads are loaded as a **Singlecellexperiment (SCE)** object by read10XCounts() of **DropletUtils** R package. A SCE object contains a **gene-by-cell count matrix**, **gene data** (gene annotation, etc) and **cell data** (sample information, experimental condition information, etc). Gene information will be stored in rowData(SCE), and cell information is stored in colData(SCE). A gene-by-cell matrix is stored as a sparse matrix in a SCE object. Ensembl gene ids is transformed into gene symbol for eaiser further analysis. In colData(sce), Sample name, cancer histologic subtypes (Diagnosis), sample ID, cell type information (from the literature) is stored.

``` r
sce <- read10xCounts(
  samples = dir,
  type="sparse",
  col.names = TRUE
)
rownames(sce) = uniquifyFeatureNames(rowData(sce)$ID, rowData(sce)$Symbol)

sce$Sample = sampleInfo$Sample
sce$Diagnosis = sampleInfo$Diagnosis
sce$ID = sampleInfo$ID
sce$Type = sampleInfo$Type

sce
```

    ## class: SingleCellExperiment 
    ## dim: 20124 5115 
    ## metadata(1): Samples
    ## assays(1): counts
    ## rownames(20124): RP11-34P13.7 FO538757.2 ... AC233755.1 AC240274.1
    ## rowData names(2): ID Symbol
    ## colnames(5115): AAACCTGAGGCGTACA-1 AAACGGGAGATCGATA-1 ...
    ##   TTTATGCTCCTTAATC-13 TTTGTCAGTTTGGGCC-13
    ## colData names(5): Sample Barcode Diagnosis ID Type
    ## reducedDimNames(0):
    ## altExpNames(0):

To remove low quality cells, several values such as number of unique molecular identifiers (UMIs) per cell, number of genes detected per cell, the percentage of UMIs assigned to mitochondrial (MT) genes are calculated using **addPerCellQC()** of **scater** R package. We define poor quality cells with &lt; 700 UMIs and &gt; 20% of UMIs assigned to MT genes and excluded them. Criteria can be visualized as histograms as below.

``` r
library(scater)

mtgenes = rowData(sce)[grep("MT-", rowData(sce)$Symbol),]$Symbol
is.mito = rownames(sce) %in% mtgenes
table(is.mito)
```

    ## is.mito
    ## FALSE  TRUE 
    ## 20111    13

``` r
sce <- addPerCellQC(
  sce,
  subsets = list(MT=mtgenes),
  percent_top = c(50, 100, 200, 500), 
  detection_limit = 5
)

sce$log10_sum = log10(sce$sum + 1)
sce$log10_detected = log10(sce$detected + 1)

umi = 700
mtpct = 20

hist(sce$sum, breaks = 100)
abline(v = umi, col="red")
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-4-1.png" style="display: block; margin: auto;" />

``` r
hist(sce$subsets_MT_percent, breaks=100)
abline(v=mtpct, col="red")
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-4-2.png" style="display: block; margin: auto;" />

``` r
filter_by_total_counts = sce$sum > umi
filter_by_mt_percent = sce$subsets_MT_percent < mtpct

sce$use <- (
  filter_by_total_counts &
    filter_by_mt_percent 
)

sce = sce[,sce$use]
```

# Basic Pipeline

After quality control, basic processes including normalization, feature selection and visualization is performed.

## Normalization

To remove cell-specific biases, cells are clustered using **quickCluster()** and cell-specific size factors are calculated using **computeSumFactors()** of **scran** R package. Raw counts of each cell are divided by cell-specific size factor and **log2-transformed** with a pseudocount of 1.

``` r
library(scran)

clusters <- quickCluster(sce)
sce <- computeSumFactors(sce, clusters = clusters)
print(summary(sizeFactors(sce)))
```

    ##     Min.  1st Qu.   Median     Mean  3rd Qu.     Max. 
    ##  0.09539  0.32598  0.54100  1.00000  1.06519 15.19534

``` r
sce.norm <- logNormCounts(sce, pseudo_count = 1)
```

## Feature Selection

To find genes contain useful information about the biology of the data, highly variable genes (HVGs) are defined by selecting the most variable genes based on their expression across cells. Genes with **&lt; 0.05 of false discovery rate (FDR)** are identified as HVGs.

``` r
dec <- modelGeneVar(sce.norm)
plot(dec$mean, dec$total, xlab="Mean log-expression", ylab="Variance")
curve(metadata(dec)$trend(x), col="blue", add=TRUE)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

``` r
hvg.norm <- getTopHVGs(dec, fdr.threshold = 0.05)
length(hvg.norm) # 551 genes
```

    ## [1] 551

## Dimension Reduction

For downstream analysis, we create Seurat object containing raw and normalized gene-by-cell count matrices. Column data (cell information) is preserved. Normalized data is scaled and principal components (PCs) are calculated by a gene-by-cell matrix with HVGs.

``` r
library(Seurat)

seurat <- as.Seurat(sce.norm,
                    counts = "counts",
                    data = "logcounts",
                    assay = "RNA")
VariableFeatures(seurat) = hvg.norm

all.genes = rownames(seurat)
seurat <- ScaleData(seurat, features = all.genes)

seurat <- RunPCA(seurat,
                 assay = "RNA",
                 npcs = 50,
                 features = hvg.norm,
                 reduction.key = "pca_",
                 verbose = FALSE)
plot((seurat@reductions$pca@stdev)^2,
     xlab = "PC",
     ylab = "Eigenvalue")
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

We set to 15 PCs for clustering and visualization. After clustering and visualization, cells are plotted in the two-dimensional TSNE or UMAP plot and cell information can be also shown.

``` r
PCA=15

seurat <- FindNeighbors(seurat, dims=1:PCA)
seurat <- FindClusters(seurat, resolution = 0.8)
```

    ## Modularity Optimizer version 1.3.0 by Ludo Waltman and Nees Jan van Eck
    ## 
    ## Number of nodes: 4749
    ## Number of edges: 155861
    ## 
    ## Running Louvain algorithm...
    ## Maximum modularity in 10 random starts: 0.9134
    ## Number of communities: 19
    ## Elapsed time: 0 seconds

``` r
seurat <- RunTSNE(seurat, dims = 1:PCA)
seurat <- RunUMAP(seurat, dims = 1:PCA)

TSNEPlot(seurat, group.by = "ID", pt.size = 1)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-8-1.png" style="display: block; margin: auto;" />

``` r
UMAPPlot(seurat, group.by = "ID", pt.size = 1)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-8-2.png" style="display: block; margin: auto;" />

## Batch Correction

On the previous TSNE (or UMAP) plot, batch effect is shown. Batch effect is removed using **RunHarmony()** in **Harmony** R package. After using RunHarmony(), it returns a Seurat object, updated with the corrected Harmony coordinates. Using the corrected Harmony coordinates, clustering and visualization are processed same as before batch correction.

``` r
library(harmony)

seurat <- RunHarmony(seurat, "ID", plot_convergence = TRUE)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-9-1.png" style="display: block; margin: auto;" />

``` r
nComp = 15
seurat <- FindNeighbors(seurat, 
                        reduction = "harmony",
                        dims=1:nComp)
seurat <- FindClusters(seurat, resolution = 0.2)
```

    ## Modularity Optimizer version 1.3.0 by Ludo Waltman and Nees Jan van Eck
    ## 
    ## Number of nodes: 4749
    ## Number of edges: 175426
    ## 
    ## Running Louvain algorithm...
    ## Maximum modularity in 10 random starts: 0.9640
    ## Number of communities: 8
    ## Elapsed time: 0 seconds

``` r
seurat <- RunTSNE(seurat,
                  reduction = "harmony",
                  dims = 1:nComp,
                  check_duplicates = FALSE)
seurat <- RunUMAP(seurat,
                  reduction = "harmony",
                  dims = 1:nComp)
```

Batch effect between patients are removed after using harmony.

``` r
DimPlot(seurat, reduction = "umap", group.by = "ID", pt.size = 1)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-10-1.png" style="display: block; margin: auto;" />

``` r
DimPlot(seurat, reduction = "tsne", group.by = "ID", pt.size = 1)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-10-2.png" style="display: block; margin: auto;" />

Also, clustering is done with the corrected Harmony coordinates.

``` r
DimPlot(seurat, reduction = "tsne", group.by = "seurat_clusters", pt.size=1, label=TRUE, label.size=10)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-11-1.png" style="display: block; margin: auto;" />

``` r
DimPlot(seurat, reduction = "umap", group.by = "seurat_clusters", pt.size=1, label=TRUE, label.size=10)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-11-2.png" style="display: block; margin: auto;" />

## Identification of cell types

Based on 'seurat\_clusters' after clustering, cells are grouped according to their cell types as annotated based on known cell lineage-specific marker genes of T cells, B cells, cancer-associated fibroblasts (CAFs), tumor-associated macrophages (TAMs), tumor-associated endothelial cells (TECs), and cells with an unknown entity but express hepatic progenitor cell markers (HPC-like).

``` r
library(ggplot2)
library(pheatmap)
library(RColorBrewer)

markers = list(
  T.cells = c("CD2", "CD3E", "CD3D", "CD3G"), #cluster 0
  B.cells = c("CD79A", "SLAMF7", "BLNK", "FCRL5"), #cluster 4,6
  TECs = c("PECAM1", "VWF", "ENG", "CDH5"), #cluster 2,7
  CAFs = c("COL1A2", "FAP", "PDPN", "DCN", "COL3A1", "COL6A1"), #cluster 1
  TAMs = c("CD14", "CD163", "CD68", "CSF1R"), #cluster 5
  HPC.like = c("EPCAM", "KRT19", "PROM1", "ALDH1A1", "CD24") #cluster 3
)
```

Expression pattern of T cell-specific marker genes are shown below.

``` r
FeaturePlot(seurat,
            features = markers$T.cells,
            order = T,
            pt.size = 1,
            ncol = 2)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

Expression pattern of B cell-specific marker genes are shown below.

``` r
FeaturePlot(seurat,
            features = markers$B.cells,
            order = T,
            pt.size = 1,
            ncol = 2)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-14-1.png" style="display: block; margin: auto;" />

Expression pattern of TEC-specific marker genes are shown below.

``` r
FeaturePlot(seurat,
            features = markers$TECs,
            order = T,
            pt.size = 1,
            ncol = 2)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

Expression pattern of CAF-specific marker genes are shown below.

``` r
FeaturePlot(seurat,
            features = markers$CAFs,
            order = T,
            pt.size = 1,
            ncol = 2)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-16-1.png" style="display: block; margin: auto;" />

Expression pattern of TAM-specific marker genes are shown below.

``` r
FeaturePlot(seurat,
            features = markers$TAMs,
            order = T,
            pt.size = 1,
            ncol = 2)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-17-1.png" style="display: block; margin: auto;" />

Expression pattern of HPC-like cell marker genes are shown below.

``` r
FeaturePlot(seurat,
            features = markers$HPC.like,
            order = T,
            pt.size = 1,
            ncol = 2)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-18-1.png" style="display: block; margin: auto;" />

Additionally, expression patterns of cell lineage-specific marker genes are shown by heatmap. By heatmap, it is possible to compare expression of marker genes between each cluster.

``` r
avgExprs <- AverageExpression(seurat,
                              features = unlist(markers),
                              assays = "RNA", slot = "data")

scaledExprs <- t(scale(t(avgExprs$RNA)))
scaledExprs[scaledExprs > -min(scaledExprs)] <- -min(scaledExprs)

palette_length = 100
my_color = colorRampPalette(rev(brewer.pal(11, "RdBu")))(palette_length)

my_breaks <- c(seq(min(scaledExprs), 0,
                   length.out=ceiling(palette_length/2) + 1),
               seq(max(scaledExprs)/palette_length,
                   max(scaledExprs),
                   length.out=floor(palette_length/2)))

pheatmap(scaledExprs,
         cluster_cols = T, cluster_rows = F, clustering_method = "ward.D2",
         treeheight_col = 0,
         breaks = my_breaks, color=my_color,
         labels_row = as.expression(lapply(rownames(scaledExprs), function(a) bquote(italic(.(a))))),
         angle_col = 315
)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-19-1.png" style="display: block; margin: auto;" />

Finally, cell types are annotated based on clusters.

``` r
seurat$celltype = seurat$seurat_clusters

seurat$celltype = gsub(0, "T.cell", seurat$celltype)
seurat$celltype = gsub(1, "CAF", seurat$celltype)
seurat$celltype = gsub(2, "TEC", seurat$celltype)
seurat$celltype = gsub(3, "HPC.like", seurat$celltype)
seurat$celltype = gsub(4, "B.cell", seurat$celltype)
seurat$celltype = gsub(5, "TAM", seurat$celltype)
seurat$celltype = gsub(6, "B.cell", seurat$celltype)
seurat$celltype = gsub(7, "TEC", seurat$celltype)

DimPlot(seurat, 
        reduction="tsne", 
        group.by="celltype",
        pt.size = 1)
```

<img src="Basic_Pipeline_files/figure-markdown_github/unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

## Session information

``` r
sessionInfo()
```

    ## R version 4.0.2 (2020-06-22)
    ## Platform: x86_64-w64-mingw32/x64 (64-bit)
    ## Running under: Windows 10 x64 (build 17763)
    ## 
    ## Matrix products: default
    ## 
    ## locale:
    ## [1] LC_COLLATE=Korean_Korea.949  LC_CTYPE=Korean_Korea.949   
    ## [3] LC_MONETARY=Korean_Korea.949 LC_NUMERIC=C                
    ## [5] LC_TIME=Korean_Korea.949    
    ## 
    ## attached base packages:
    ## [1] parallel  stats4    stats     graphics  grDevices utils     datasets 
    ## [8] methods   base     
    ## 
    ## other attached packages:
    ##  [1] RColorBrewer_1.1-2          pheatmap_1.0.12            
    ##  [3] harmony_1.0                 Rcpp_1.0.6                 
    ##  [5] SeuratObject_4.0.0          Seurat_4.0.0               
    ##  [7] scran_1.16.0                scater_1.16.2              
    ##  [9] ggplot2_3.3.3               dplyr_1.0.4                
    ## [11] DropletUtils_1.8.0          SingleCellExperiment_1.12.0
    ## [13] SummarizedExperiment_1.20.0 Biobase_2.50.0             
    ## [15] GenomicRanges_1.42.0        GenomeInfoDb_1.26.2        
    ## [17] IRanges_2.24.1              S4Vectors_0.28.1           
    ## [19] BiocGenerics_0.36.0         MatrixGenerics_1.2.1       
    ## [21] matrixStats_0.58.0         
    ## 
    ## loaded via a namespace (and not attached):
    ##   [1] plyr_1.8.6                igraph_1.2.6             
    ##   [3] lazyeval_0.2.2            splines_4.0.2            
    ##   [5] BiocParallel_1.22.0       listenv_0.8.0            
    ##   [7] scattermore_0.7           digest_0.6.27            
    ##   [9] htmltools_0.5.1.1         viridis_0.5.1            
    ##  [11] magrittr_2.0.1            tensor_1.5               
    ##  [13] cluster_2.1.0             ROCR_1.0-11              
    ##  [15] limma_3.44.3              globals_0.14.0           
    ##  [17] R.utils_2.10.1            colorspace_2.0-0         
    ##  [19] ggrepel_0.9.1             xfun_0.20                
    ##  [21] crayon_1.4.0              RCurl_1.98-1.2           
    ##  [23] jsonlite_1.7.2            spatstat_1.64-1          
    ##  [25] spatstat.data_1.7-0       survival_3.1-12          
    ##  [27] zoo_1.8-8                 glue_1.4.2               
    ##  [29] polyclip_1.10-0           gtable_0.3.0             
    ##  [31] zlibbioc_1.36.0           XVector_0.30.0           
    ##  [33] leiden_0.3.7              DelayedArray_0.16.1      
    ##  [35] BiocSingular_1.4.0        Rhdf5lib_1.10.1          
    ##  [37] future.apply_1.7.0        HDF5Array_1.16.1         
    ##  [39] abind_1.4-5               scales_1.1.1             
    ##  [41] DBI_1.1.1                 edgeR_3.30.3             
    ##  [43] miniUI_0.1.1.1            viridisLite_0.3.0        
    ##  [45] xtable_1.8-4              reticulate_1.18          
    ##  [47] dqrng_0.2.1               rsvd_1.0.3               
    ##  [49] htmlwidgets_1.5.3         httr_1.4.2               
    ##  [51] ellipsis_0.3.1            ica_1.0-2                
    ##  [53] farver_2.0.3              pkgconfig_2.0.3          
    ##  [55] R.methodsS3_1.8.1         uwot_0.1.10              
    ##  [57] deldir_0.2-9              locfit_1.5-9.4           
    ##  [59] labeling_0.4.2            tidyselect_1.1.0         
    ##  [61] rlang_0.4.10              reshape2_1.4.4           
    ##  [63] later_1.1.0.1             munsell_0.5.0            
    ##  [65] tools_4.0.2               generics_0.1.0           
    ##  [67] ggridges_0.5.3            evaluate_0.14            
    ##  [69] stringr_1.4.0             fastmap_1.1.0            
    ##  [71] yaml_2.2.1                goftest_1.2-2            
    ##  [73] knitr_1.31                fitdistrplus_1.1-3       
    ##  [75] purrr_0.3.4               RANN_2.6.1               
    ##  [77] pbapply_1.4-3             future_1.21.0            
    ##  [79] nlme_3.1-148              mime_0.9                 
    ##  [81] R.oo_1.24.0               compiler_4.0.2           
    ##  [83] beeswarm_0.2.3            plotly_4.9.3             
    ##  [85] png_0.1-7                 spatstat.utils_2.0-0     
    ##  [87] tibble_3.0.6              statmod_1.4.35           
    ##  [89] stringi_1.5.3             highr_0.8                
    ##  [91] RSpectra_0.16-0           lattice_0.20-41          
    ##  [93] Matrix_1.2-18             vctrs_0.3.6              
    ##  [95] pillar_1.4.7              lifecycle_0.2.0          
    ##  [97] lmtest_0.9-38             RcppAnnoy_0.0.18         
    ##  [99] BiocNeighbors_1.6.0       data.table_1.13.6        
    ## [101] cowplot_1.1.1             bitops_1.0-6             
    ## [103] irlba_2.3.3               httpuv_1.5.5             
    ## [105] patchwork_1.1.1           R6_2.5.0                 
    ## [107] promises_1.1.1            KernSmooth_2.23-18       
    ## [109] gridExtra_2.3             vipor_0.4.5              
    ## [111] parallelly_1.23.0         codetools_0.2-16         
    ## [113] MASS_7.3-51.6             assertthat_0.2.1         
    ## [115] rhdf5_2.32.4              withr_2.4.1              
    ## [117] sctransform_0.3.2         GenomeInfoDbData_1.2.4   
    ## [119] mgcv_1.8-31               grid_4.0.2               
    ## [121] rpart_4.1-15              tidyr_1.1.2              
    ## [123] rmarkdown_2.6             DelayedMatrixStats_1.10.1
    ## [125] Rtsne_0.15                shiny_1.6.0              
    ## [127] ggbeeswarm_0.6.0

## References

L. Ma, M.O. Hernandez, Y. Zhao, M. Mehta, B. Tran, M. Kelly, Z. Rae, J.M. Hernandez, J.L. Davis, S.P. Martin, D.E. Kleiner, S.M. Hewitt, K. Ylaya, B.J. Wood, T.F. Greten, X.W. Wang. Tumor cell biodiversity drives microenvironmental reprogramming in liver cancer. Canc. Cell, 36 (4): 418-430 (2019) Lun, A. T. L. et al. EmptyDrops: distinguishing cells from empty droplets in droplet-based single-cell RNA sequencing data. Genome Biol. 20, 63 (2019) McCarthy, D. J., Campbell, K. R., Lun, A. T. & Wills, Q. F. Scater: pre-processing, quality control, normalization and visualization of single-cell RNA-seq data in R. Bioinformatics 33, 1179–1186 (2017) Lun, A. T., McCarthy, D. J. & Marioni, J. C. A step-by-step workflow for low-level analysis of single-cell RNA-seq data with Bioconductor. F1000Res 5, 2122 (2016). Butler, A., Hoffman, P., Smibert, P., Papalexi, E. & Satija, R. Integrating single-cell transcriptomic data across different conditions, technologies, and species. Nat. Biotechnol. 36, 411–420 (2018). Korsunsky, I., Millard, N., Fan, J. et al. Fast, sensitive and accurate integration of single-cell data with Harmony. Nat Methods 16, 1289–1296 (2019).