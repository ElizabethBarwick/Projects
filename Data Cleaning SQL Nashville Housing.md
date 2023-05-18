'''sql
-- Cleaning Data in SQL Queries

-- This file shows Nashville Housing data including sale dates, prices, owners for different parcel IDs and property addresses.
-- The following SQL code is used to clean and prepare the data so that it is in a more consistent and usable format.

-- View the initial data file
SELECT *
FROM dbo.NashvilleHousing

-- Standardize Date Format by removing the time 
SELECT SaleDate
FROM dbo.NashvilleHousing

UPDATE NashvilleHousing
SET SaleDate = CONVERT(Date, SaleDate)

-- Create new column for converted date--------------------------------------
ALTER TABLE NashvilleHousing
ADD SaleDateConverted Date;

UPDATE NashvilleHousing
SET SaleDateConverted = CONVERT(Date, SaleDate); 

-- Populate Property Address Data----------------------------------------------
-- While there may be multiple listings for a given ParcelID, some contain a PropertyAddress and others are null
-- In viewing other records, ParcelID and PropertyAddress have a one-to-one correspondence
-- Replace any nulls in PropertyAddress with the address from another record of same ParcelID
-- Use a self join to match ParcelIDs where one of the PropertyAddresses is null
SELECT a.ParcelID, a.PropertyAddress, b.ParcelID, b.PropertyAddress, ISNULL(a.PropertyAddress, b.PropertyAddress)
FROM dbo.NashvilleHousing a
JOIN dbo.NashvilleHousing b
	ON a.ParcelID=b.ParcelID
	AND a.UniqueID <> b.UniqueID
WHERE a.PropertyAddress IS NULL

UPDATE a
SET PropertyAddress = ISNULL(a.PropertyAddress,b.PropertyAddress)
FROM dbo.NashvilleHousing a
JOIN dbo.NashvilleHousing b
	ON a.ParcelID=b.ParcelID
	AND a.UniqueID <> b.UniqueID
WHERE a.PropertyAddress IS NULL

SELECT *
FROM dbo.NashvilleHousing
ORDER BY ParcelID

-- Breaking out PropertyAddress into Individual Columns-------------------------------
-- Current PropertyAddress field contains street address as well as city
-- Separate out substrings for street address and city
SELECT PropertyAddress
FROM dbo.NashvilleHousing

SELECT 
	SUBSTRING(PropertyAddress, 1, CHARINDEX(',',PropertyAddress)-1) AS Address,
	SUBSTRING(PropertyAddress, CHARINDEX(',',PropertyAddress)+1, LEN(PropertyAddress)) AS City
FROM dbo.NashvilleHousing

-- Create new columns for Address and City--------------------------------------
ALTER TABLE NashvilleHousing
ADD PropertySplitAddress NVARCHAR(255);

UPDATE NashvilleHousing
SET PropertySplitAddress = SUBSTRING(PropertyAddress, 1, CHARINDEX(',',PropertyAddress)-1); 

ALTER TABLE NashvilleHousing
ADD PropertySplitCity NVARCHAR(255);

UPDATE NashvilleHousing
SET PropertySplitCity = SUBSTRING(PropertyAddress, CHARINDEX(',',PropertyAddress)+1, LEN(PropertyAddress)); 

-- Break OwnerAddress into Individual Columns
-- Current OwnerAddress field contains street address, city and state
-- Parse OwnerAddress into three separate parts for street address, city and state
SELECT OwnerAddress
FROM dbo.NashvilleHousing

SELECT
	PARSENAME(REPLACE(OwnerAddress,',','.'), 3),
	PARSENAME(REPLACE(OwnerAddress,',','.'), 2),
	PARSENAME(REPLACE(OwnerAddress,',','.'), 1)
FROM dbo.NashvilleHousing

-- Create new columns for owner address--------------------------------------
ALTER TABLE NashvilleHousing
ADD OwnerSplitAddress NVARCHAR(255);

UPDATE NashvilleHousing
SET  OwnerSplitAddress = PARSENAME(REPLACE(OwnerAddress,',','.'), 3); 

ALTER TABLE NashvilleHousing
ADD  OwnerSplitCity NVARCHAR(255);

UPDATE NashvilleHousing
SET  OwnerSplitCity= PARSENAME(REPLACE(OwnerAddress,',','.'), 2); 

ALTER TABLE NashvilleHousing
ADD  OwnerSplitState NVARCHAR(255);

UPDATE NashvilleHousing
SET  OwnerSplitState = PARSENAME(REPLACE(OwnerAddress,',','.'), 1); 


-- In the SoldAsVacant field, there are inconsistences in data
-- Some entered as Y, Yes, N, No
-- Change Y and N to Yes and No-----------------------------------------------

SELECT DISTINCT(SoldAsVacant), COUNT(SoldAsVacant)
FROM dbo.NashvilleHousing
GROUP BY SoldAsVacant
ORDER BY 2

SELECT SoldAsVacant,
CASE WHEN SoldAsVacant = 'Y' THEN 'Yes'
	WHEN SoldAsVacant = 'N' THEN 'No'
	ELSE SoldAsVacant
	END
FROM dbo.NashvilleHousing

UPDATE NashvilleHousing
SET SoldAsVacant = CASE WHEN SoldAsVacant = 'Y' THEN 'Yes'
	WHEN SoldAsVacant = 'N' THEN 'No'
	ELSE SoldAsVacant
	END

-- Remove Duplicates------------------------------------------------------------
-- There are duplicate rows that contain same ParcelID, PropertyAddress, SalePrice, SaleDate and LegalReference
-- Use row number to partition over these columns
-- Delete row numbers greater than 1 to remove duplicates
WITH RowNumCTE AS (
	SELECT *,
		ROW_NUMBER() OVER(
		PARTITION BY ParcelID,
		PropertyAddress,
		SalePrice,
		SaleDate,
		LegalReference
		ORDER BY UniqueID
		) row_num
	FROM dbo.NashvilleHousing
	)

-- DELETE
SELECT *
FROM RowNumCTE
WHERE row_num>1

------------------------------------
-- Delete Unused Columns------------------------------------------------------------------
-- After splitting certain fields into more usable formats, delete original columns that are no longer needed

SELECT *
FROM dbo.NashvilleHousing
ORDER BY ParcelID

ALTER TABLE dbo.NashvilleHousing
DROP COLUMN OwnerAddress, TaxDistrict, PropertyAddress

ALTER TABLE dbo.NashvilleHousing
DROP COLUMN SaleDate
'''
