# Layoffs Data Cleaning and EDA SQL Scripts

## Overview
This repository contains SQL scripts for cleaning, standardizing, and exploring a layoffs dataset. The data processing includes:

1. Removing duplicates
2. Standardizing data
3. Handling null or blank values
4. Removing unnecessary columns
5. Exploratory Data Analysis (EDA)

## SQL Data Cleaning Steps

### 1. Remove Duplicates
- Create a staging table for transformations:
  ```sql
  CREATE TABLE layoffs_staging LIKE layoffs;
  INSERT layoffs_staging SELECT * FROM layoffs;
  ```
- Identify duplicate records:
  ```sql
  WITH duplicate_cte AS (
      SELECT *,
             ROW_NUMBER() OVER (
                 PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
             ) AS row_num 
      FROM layoffs_staging
  )
  SELECT * FROM duplicate_cte WHERE row_num > 1;
  ```
- Remove duplicates by inserting data into a new table:
  ```sql
  CREATE TABLE layoffs_staging2 (
      company TEXT,
      location TEXT,
      industry TEXT,
      total_laid_off INT DEFAULT NULL,
      percentage_laid_off TEXT,
      `date` TEXT,
      stage TEXT,
      country TEXT,
      funds_raised_millions INT DEFAULT NULL,
      row_num INT
  );
  
  INSERT INTO layoffs_staging2
  SELECT *,
         ROW_NUMBER() OVER (
             PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions
         ) AS row_num 
  FROM layoffs_staging;
  
  DELETE FROM layoffs_staging2 WHERE row_num > 1;
  ```

### 2. Standardizing the Data
- Trim company names:
  ```sql
  UPDATE layoffs_staging2 SET company = TRIM(company);
  ```
- Standardize industry names:
  ```sql
  UPDATE layoffs_staging2 SET industry = 'Crypto' WHERE industry LIKE 'Crypto%';
  ```
- Convert date format:
  ```sql
  ALTER TABLE layoffs_staging2 MODIFY COLUMN `date` DATE;
  ```

### 3. Handling Null or Blank Values
- Identify missing values:
  ```sql
  SELECT * FROM layoffs_staging2 WHERE total_laid_off IS NULL AND percentage_laid_off IS NULL;
  ```
- Replace empty industry values with NULL and update them using existing data:
  ```sql
  UPDATE layoffs_staging2 SET industry = NULL WHERE industry = '';
  
  UPDATE layoffs_staging2 t1
  JOIN layoffs_staging2 t2
  ON t1.company = t2.company
  SET t1.industry = t2.industry
  WHERE (t1.industry IS NULL OR t1.industry = '') AND t2.industry IS NOT NULL;
  ```

### 4. Remove Unnecessary Columns
- Drop the `row_num` column:
  ```sql
  ALTER TABLE layoffs_staging2 DROP COLUMN row_num;
  ```

## Exploratory Data Analysis (EDA)

### 1. Overview of the Dataset
- Display all records:
  ```sql
  SELECT * FROM layoffs_staging2;
  ```
- Identify the maximum layoffs:
  ```sql
  SELECT MAX(total_laid_off) FROM layoffs_staging2;
  ```
- Identify the maximum layoffs and percentage laid off:
  ```sql
  SELECT MAX(total_laid_off), MAX(percentage_laid_off) FROM layoffs_staging2;
  ```

### 2. Company and Industry Insights
- Companies with the highest layoffs:
  ```sql
  SELECT company, SUM(total_laid_off) FROM layoffs_staging2 GROUP BY company ORDER BY 2 DESC;
  ```
- Industries with the highest layoffs:
  ```sql
  SELECT industry, SUM(total_laid_off) FROM layoffs_staging2 GROUP BY industry ORDER BY 2 DESC;
  ```

### 3. Country and Date Analysis
- Countries with the highest layoffs:
  ```sql
  SELECT country, SUM(total_laid_off) FROM layoffs_staging2 GROUP BY country ORDER BY 2 DESC;
  ```
- Date range of layoffs:
  ```sql
  SELECT MIN(`date`), MAX(`date`) FROM layoffs_staging2;
  ```
- Layoffs per year:
  ```sql
  SELECT YEAR(`date`), SUM(total_laid_off) FROM layoffs_staging2 GROUP BY YEAR(`date`) ORDER BY 1 DESC;
  ```

### 4. Monthly and Rolling Layoff Trends
- Monthly layoffs:
  ```sql
  SELECT SUBSTRING(`date`,1,7) AS `MONTH`, SUM(total_laid_off) FROM layoffs_staging2 WHERE SUBSTRING(`date`,1,7) IS NOT NULL GROUP BY `MONTH` ORDER BY 1 ASC;
  ```
- Rolling total layoffs:
  ```sql
  WITH Rolling_Total AS (
      SELECT SUBSTRING(`date`,1,7) AS `MONTH`, SUM(total_laid_off) AS total_off FROM layoffs_staging2 WHERE SUBSTRING(`date`,1,7) IS NOT NULL GROUP BY `MONTH` ORDER BY 1 ASC
  )
  SELECT `MONTH`, total_off, SUM(total_off) OVER (ORDER BY `MONTH`) AS rolling_total FROM Rolling_Total;
  ```

### 5. Top Companies by Year
- Companies with the highest layoffs per year:
  ```sql
  WITH Company_Year (company, years, total_laid_off) AS (
      SELECT company, YEAR(`date`), SUM(total_laid_off) FROM layoffs_staging2 GROUP BY company, YEAR(`date`)
  ), Company_Year_Rank AS (
      SELECT *, DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS Ranking FROM Company_Year WHERE years IS NOT NULL
  )
  SELECT * FROM Company_Year_Rank WHERE Ranking <= 5;
  ```

## Usage
- Execute each step in order to clean and analyze the dataset.
- Use the final cleaned table (`layoffs_staging2`) for further analysis and reporting.

## License
This project is open-source and available for modification and distribution.

---
Feel free to contribute or suggest improvements!

