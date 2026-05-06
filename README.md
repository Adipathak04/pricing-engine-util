# pricing-engine-util

/*****************************************************************
Author - Shivam D, Chirag C
Date of Creation - 10 May 2024
Change History :
Last Updated By Last Update Date
Shivam Debnath 17 May 2024
******************************************************************/
returnJson = json();
retstr = stringbuilder();
parentDict = dict("string");
invokeAction = jsonget(requestJson, "invokeAction", "string", "");
bomLevel2DocNumber = "";
parentDocNumber = "";
totalPriceDict = dict("string"); //Added for proposals
tierProposalJson = json();
projectedTCV = 0.0;
annualContractTerm = 0;
// Price Calculation start
if (invokeAction == "PriceCalc") {
    assumptionDataStr = jsonget(requestJson, "finalAssumptionStr", "string", "");
    lineDataJson = jsonget(requestJson, "lineDataJson", "json", json());
    pricingJson = jsonget(requestJson, "pricingJson", "json", json());
    finalPriceJson = jsonget(requestJson, "finalPriceJson", "json", json());
    docNumArr = jsonkeys(lineDataJson);
    formulaSKUJson = jsonget(requestJson, "formulaSKUJson", "json", json());
    cQFSKUsArr = split(jsonget(requestJson, "cQFSKUs", "string", ""), "#");
    multiParamFormulaJson = jsonget(requestJson, "multiParamFormulaJson", "json", json());
    variableUnitRateSKUJson = jsonget(requestJson, "variableUnitRateSKUJson", "json", json());
    modelContractStartDateJson = jsonget(requestJson, "modelContractStartDateJson", "json", json());
    variableUnitRateSKUsArr = jsonkeys(variableUnitRateSKUJson);
    attributeJson = jsonget(requestJson, "attributeJson", "json", json());
    //txnType = jsonget(attributeJson, "transactionType_t_c", "string", ""); // added by Shivam
    coreProduct = jsonget(attributeJson, "coreProduct", "string", ""); // Added : Bibin Sajeevan : 07 Feb 2025 : Adding for Ad Hoc Packageing
    adHocCoreType = jsonget(attributeJson, "adHocCoreType", "string", ""); // Added : Bibin Sajeevan : 07 Feb 2025 : Adding for Ad Hoc Packageing
    tieredMinJson = jsonget(requestJson, "tieredMinJson", "json", json());
    productCalcPatternDetailJson = jsonget(requestJson, "productCalcPatternDetailJson", "json", json());
    oneTimeSKUAssociationJson = jsonget(requestJson, "oneTimeSKUAssociationJson", "json", json()); //Navya: Added this for billing integration
    specialTCVInfoJson = jsonget(requestJson, "specialTCVInfoJson", "json", json());
    signingBonusSkuJson = jsonget(requestJson, "signingBonusSkuJson", "json", json()); //Sateesh : Added for Signing bonus new pattern 18 Aug 2025
    variablepricingSKuJsonforGrpMin = jsonget(requestJson, "variablepricingSKuJsonforGrpMin", "json", json()); //Added by Basavaraj on 24-MAR-2026 for ARO Bundle SKU's to display unit rate but to include the actual quantity to multiply for group minimum
    /*Reordering variable unit rate SKU to make them execute at last : Start*/
    if (jsontostr(variableUnitRateSKUJson) <> "{}") {
        variableUnitRateSKUDocNumJsonArr = jsonpathgetmultiple(variableUnitRateSKUJson, "$..docNum");
        variableUnitRateSKUDocNumSizeArr = range(jsonarraysize(variableUnitRateSKUDocNumJsonArr));
        for each in variableUnitRateSKUDocNumSizeArr {
            variableUnitRateSKUDocNum = jsonarrayget(variableUnitRateSKUDocNumJsonArr, each);
            if (findinarray(docNumArr, variableUnitRateSKUDocNum) >= 0) {
                remove(docNumArr, findinarray(docNumArr, variableUnitRateSKUDocNum));
                append(docNumArr, variableUnitRateSKUDocNum);
            }
        }
    }
    /*Reordering variable unit rate SKU to make them execute at last : End*/
    /*Reordering Formula SKU to make them execute at last : Start*/
    if (jsontostr(formulaSKUJson) <> "{}") {
        formulaSkuDocNumJsonArr = jsonpathgetmultiple(formulaSKUJson, "$..docNum");
        formulaSkuDocNumSizeArr = range(jsonarraysize(formulaSkuDocNumJsonArr));
        for each in formulaSkuDocNumSizeArr {
            formulaSkuDocNum = jsonarrayget(formulaSkuDocNumJsonArr, each);
            if (findinarray(docNumArr, formulaSkuDocNum) >= 0) {
                remove(docNumArr, findinarray(docNumArr, formulaSkuDocNum));
                append(docNumArr, formulaSkuDocNum);
            }
        }
    }
    /*Reordering Formula SKU to make them execute at last : End*/
    /*Reordering MultiParamFormula SKU to make them execute at last : Start*/
    if (jsontostr(multiParamFormulaJson) <> "{}") {
        multiParamFormulaSkuDocNumJsonArr = jsonpathgetmultiple(multiParamFormulaJson, "$..docNum");
        multiParamFormulaSkuDocNumSizeArr = range(jsonarraysize(multiParamFormulaSkuDocNumJsonArr));
        for each in multiParamFormulaSkuDocNumSizeArr {
            multiParamformulaSkuDocNum = jsonarrayget(multiParamFormulaSkuDocNumJsonArr, each);
            if (findinarray(docNumArr, multiParamformulaSkuDocNum) >= 0) {
                remove(docNumArr, findinarray(docNumArr, multiParamformulaSkuDocNum));
                append(docNumArr, multiParamformulaSkuDocNum);
            }
        }
    }
    /*Reordering MultiParamFormula SKU to make them execute at last : End*/
    groupMinimunJson = json();
    multiGroupMinJson = json(); //When a group minimum is contributing to another group minimum
    minimumCalcJson = json();
    aggregatePricingJson = json();
    needsProductSpecialistInput = false;
    miscQuote = false; // Prod Issue on misc fee requiring quote
    rollUpSKUJson = json();
    if (sizeofarray(docNumArr) > 0) {
        //Loop on each document number
        for each in docNumArr {
            isListPriceNA = false;
            quoteNeeded = false;
            salesQuoteNeeded = false;
            hasListPrice = false;
            hasUnitRate = false;
            isTierPricingSKU = false;
            isAggregatePricingSKU = false;
            isGroupMinimumContributor = false;
            isMultiTierDisplay = false;
            nextLowerBound = 0;
            priceLabel = "";
            dynamicPriceLabel = "";
            displayMinPrice = "";
            tieredMinimumJsonArray = jsonarray();
            skuNumJson = jsonget(lineDataJson, each, "json", json());
            skuNum = jsonget(skuNumJson, "partNum", "string", "");
            skuFormulaJson = jsonget(formulaSKUJson, skuNum, "json", json());
            skuDesc = jsonget(skuNumJson, "desc", "string", "");
            propDisplayFormat = jsonget(skuNumJson, "propDisplayFormat", "string", "");
            changedQty = 0.0;
            userInputPrice = jsonget(skuNumJson, "userInputPrice", "float", 0.0);
            userInputMinPrice = jsonget(skuNumJson, "userInputMinPrice", "float", 0.0);
            previousQtyFactor = jsonget(skuNumJson, "GraduatedPreviousQty", "float", 0.0); //Only in case of graduated Calcualtion this will come.
            modelDocNum = jsonget(skuNumJson, "modelDocNumber", "string", "");
            lineData = jsonget(skuNumJson, "lineData", "string", "");
            customDiscountType = jsonget(skuNumJson, "customDiscountType", "string", "");
            customDiscountValue = jsonget(skuNumJson, "customDiscountValue", "string", "");
            minimumDiscountType = jsonget(skuNumJson, "minimumDiscountType", "string", "");
            minimumDiscountValue = jsonget(skuNumJson, "minimumDiscountValue", "float", 0.0);
            tCVQtyAttr = jsonget(skuNumJson, "TCVQtyAttr", "float", 0.0);
            tCVTierIDAttr = jsonget(skuNumJson, "tCVTierIDAttr", "float", 0.0);
            tierUpdated = jsonget(skuNumJson, "tierUpdated", "boolean", false);
            tierDiscountApplied = jsonget(skuNumJson, "isTierDiscountApplied", "boolean", false);
            contractStartDate = jsonget(skuNumJson, "contractStartDate", "string", "");
            contractEndDate = jsonget(skuNumJson, "contractEndDate", "string", "");
            aPEPercent = jsonget(skuNumJson, "aPEPercent", "string", "");
            contractTerm = jsonget(skuNumJson, "contractTerm", "string", "");
            aBOActionCode = jsonget(skuNumJson, "ABOActionCode", "string", "");
            skuDisplayName = jsonget(skuNumJson, "skuDisplayName", "string", "");
            modelVarName = jsonget(skuNumJson, "modelVarName", "string", "");
            txnType = jsonget(skuNumJson, "quoteType_l", "string", "");
            adHocIncludedOrWaivedBool = jsonget(skuNumJson, "adHocIncludedOrWaivedBool", "boolean", false); //Added : Bibin Sajeevan : 26 Feb 2025 : For Ad Hoc Packages
            //Initilaised below to consider Cost and Margin for Price Calculation of Managed Solution Products - Chandu.P - 3/25/2026
            cost = 0.0;
            margin = 0.0;
            if (findinarray(jsonkeys(skuNumJson), "AnnualContractTerm") > -1) {
                annualContractTerm = jsonget(skuNumJson, "AnnualContractTerm", "integer", 0); // Added by Chandu Pendyala for contract term changes when Fee type and Billing Frequency is Years 9/23/2025
            } else {
                //Setting the Annual contract term in the scenario where Annual contract term is not a user input. - Chandu Pendyala - Paymon and REGU
                if (isnumber(contractTerm)) {
                    annualContractTerm = atoi(contractTerm) / 12;
                }
            }

            actionCodeCRM = jsonget(skuNumJson, "actionCodeforCRM", "string", ""); // added by Sateesh Somisetti

            if (actionCodeCRM <> "") {
                sbappend(retstr, each, "~actionCodeForCRM_l_c~", actionCodeCRM, "|");
            }

            //added to mark action code for crm as new when new product is added in migration quote - 05/14
            if (jsonget(skuNumJson, "bSol_newConfiguration_pf", "boolean", false)) {
                sbappend(retstr, each, "~actionCodeForCRM_l_c~New|");
                sbappend(retstr, modelDocNum, "~actionCodeForCRM_l_c~New|");
            }

            if (modelVarName <> "") {
                aCVTCVPercentContriMonthly = jsonpathgetsingle(productCalcPatternDetailJson, "$." + modelVarName + ".ACVTCVPercentMonthly", "float", 100.0);
                aCVTCVPercentContriImpl = jsonpathgetsingle(productCalcPatternDetailJson, "$." + modelVarName + ".ACVTCVPercentImpl", "float", 100.0);
                aCVTCVPercentContriLicense = jsonpathgetsingle(productCalcPatternDetailJson, "$." + modelVarName + ".ACVTCVPercentLicense", "float", 100.0);
            } else {
                aCVTCVPercentContriMonthly = 100.0;
                aCVTCVPercentContriImpl = 100.0;
                aCVTCVPercentContriLicense = 100.0;
            }
            invoiceMatchRate = jsonget(skuNumJson, "invoiceMatchRate", "float", 0.0); // for renewal
            invoiceMatchRateMinimum = jsonget(skuNumJson, "invoiceMatchRateMinimum", "float", 0.0); // for renewal
            renew = jsonget(skuNumJson, "renew", "boolean", false); // for renewal
            invoiceMatchRateListPrice = 0.0;

            //if action code is Add then only calculate prices
            aboJson = json();
            if (jsonget(skuNumJson, "aBOJson_l_c", "string", "") <> "") {
                aboJson = jsonget(skuNumJson, "aBOJson_l_c", "json", json());
            }
            minimumCalcType = "";
            newMinimumVal1 = 0.0;
            minimumRunRateListPrice = 0.0;
            userInputListPriceCheck = false;
            userTypeInputListPriceCheck = "";
            unitRateForAggregate = 0.0;

            //Added for Proposal
            lineBomLevel = jsonget(skuNumJson, "lineBomLevel", "string", "");
            parentDocNumber = jsonget(skuNumJson, "parentDocNumber", "string", "");
            if (lineBomLevel == "2") {
                bomLevel2DocNumber = jsonget(skuNumJson, "modelDocNumber", "string", "");
            }

            lineLevelTextAreaJson = json();
            if (lineData <> "") {
                lineLevelTextAreaJson = json(lineData);
            }
            if (jsonpathcheck(pricingJson, "$." + skuNum)) {
                minApplying = false;
                minValue = 0.0;
                minValueBeforeDiscount = 0.0;
                minValueDiscountPercent = 0.0;
                proposalTierDisplayDataForProposal = jsonarray();
                multiplyFactor = jsonget(skuNumJson, "MultiplyFactor", "float", 1.0);
                PrevQtySetFlag = jsonget(skuNumJson, "PrevQtySetFlag", "boolean", false); //used to check if the previous quantity is already set in Pricing Library
                minTierIdentifier = jsonget(skuNumJson, "MinTierIdentifier", "integer", 1);
                typeOfTier = jsonpathgetsingle(pricingJson, "$." + skuNum + ".TypeOfTier", "string", "");
                typeOfPricing = jsonpathgetsingle(pricingJson, "$." + skuNum + ".TypeOfPricing", "string", "");
                uOM = jsonpathgetsingle(pricingJson, "$." + skuNum + ".UOM", "string", "");
                tierCalc = jsonpathgetsingle(pricingJson, "$." + skuNum + ".TierCalc", "string", "");
                tierDataJsonArr = jsonpathgetsingle(pricingJson, "$." + skuNum + ".TierData", "jsonarray", jsonarray());
                realTimeTierData = jsonget(skuNumJson, "proposalTierDisplay", "jsonarray", jsonarray());
                realTimeMinTierData = jsonget(skuNumJson, "minTierDisplay", "jsonarray", jsonarray());
                minTierUpdated = jsonget(skuNumJson, "minTierUpdated", "boolean", false);
                tierDiscountApplicable = jsonpathgetsingle(pricingJson, "$." + skuNum + ".TierDiscApplicable", "string", "");
                minimumCalcType = jsonpathgetsingle(pricingJson, "$." + skuNum + ".MinimumCalcType", "string", "");
                runrateRounding = jsonpathgetsingle(pricingJson, "$." + skuNum + ".RunrateRounding", "integer", 0);
                aggrRollupSKU = jsonpathgetsingle(pricingJson, "$." + skuNum + ".AggrRollupSKU", "string", "");
                MinimumCalcGrpSKUID = jsonpathgetsingle(pricingJson, "$." + skuNum + ".MinimumCalcGrpSKUID", "string", "");
                feeType = jsonpathgetsingle(pricingJson, "$." + skuNum + ".FeeType", "string", "");
                billingFrequency = jsonpathgetsingle(pricingJson, "$." + skuNum + ".BillingFrequency", "string", "");
                //JIRA 2949 : Start : Build a setup where contract start and end date can be adjusted. There are 2 SKU's, one will charged for first year and second one will be for rest of all the following year.
                if (lower(feeType) == "rebate"
                    AND modelDocNum <> "") {
                    contractStartDate = jsonget(modelContractStartDateJson, modelDocNum, "string", "");
                    contractStartMonth = jsonpathgetsingle(pricingJson, "$." + skuNum + ".ContractStartMonth", "integer", 1);
                    contractEndMonth = jsonpathgetsingle(pricingJson, "$." + skuNum + ".ContractEndMonth", "integer", 999999999); //999999999 months is the max a data table can store
                    if (contractEndMonth == 0) {
                        contractEndMonth = 999999999;
                    }
                    if (isnumber(contractTerm)) {
                        contractTerm_integer = atoi(contractTerm);
                    }
                    //Check the contract month of a SKU and deciding the contract start and end date for the SKU, this changes will play role in ACV TCV also
                    if (contractStartMonth > 0) {
                        contractStartDate_date = strtojavadate(contractStartDate, "MM/dd/yyyy");
                        contractStartDate_date = addmonths(contractStartDate_date, contractStartMonth);
                        contractStartDate = datetostr(contractStartDate_date);
                        if (contractStartMonth >= 12) {
                            sbappend(retstr, each, "~aCVExclusionFlag_l_c~true", "|"); // Exclude From ACV Based On Contract Start Date, this SKU will only participate in TCV
                        }
                    } else {
                        sbappend(retstr, each, "~aCVExclusionFlag_l_c~false", "|"); // Exclude From ACV Based On Contract Start Date, this SKU will only participate in TCV
                    }
                    if (contractEndMonth <= contractTerm_integer) { //if contract term in current quote is higher or equal to the contractEndMonth stored in data table then use below calculation
                        contractStartDate_date = strtojavadate(contractStartDate, "MM/dd/yyyy");
                        contractEndDate_date = addmonths(contractStartDate_date, contractEndMonth);
                        contractEndDate = datetostr(contractEndDate_date);
                        contractTerm = string(contractEndMonth - contractStartMonth);

                        /*if (lower(feeType) == "rebate"
                            AND lower(billingFrequency) == "years") { //Added as part of Rebate,Annual pricing 2/05/2025
                            contractTerm = string(atoi(contractTerm) / 12);
                        }*/

                        sbappend(retstr, each, "~contractTerm_l_c~", contractTerm, "|");
                    }
                    elif(contractEndMonth > contractTerm_integer) { //if contract term in current quote is lesser then the contractEndMonth stored in data table then use below calculation
                        contractStartDate_date = strtojavadate(contractStartDate, "MM/dd/yyyy");
                        contractEndDate_date = addmonths(contractStartDate_date, contractTerm_integer - contractStartMonth);
                        contractEndDate = datetostr(contractEndDate_date);
                        contractTerm = string(contractTerm_integer - contractStartMonth);
                        sbappend(retstr, each, "~contractTerm_l_c~", contractTerm, "|");
                    }
                    sbappend(retstr, each, "~contractStartDate_l~", contractStartDate, "|");
                    sbappend(retstr, each, "~contractEndDate_l~", contractEndDate, "|");
                }

                // Added by Chandu Pendyala for contract term changes when Fee type and Billing Frequency is Years 9/23/2025
                if (lower(feeType) == "years"
                    OR(lower(feeType) == "license"
                        AND lower(billingFrequency) == "years")) {
                    contractStartDate = jsonget(modelContractStartDateJson, modelDocNum, "string", "");
                    contractStartMonth = jsonpathgetsingle(pricingJson, "$." + skuNum + ".ContractStartMonth", "integer", 1);
                    contractEndMonth = jsonpathgetsingle(pricingJson, "$." + skuNum + ".ContractEndMonth", "integer", 999999999); //999999999 months is the max a data table can store
                    if (contractEndMonth == 0) {
                        contractEndMonth = 999999999;
                    }
                    if (isnumber(contractTerm)) {
                        contractTerm_integer = atoi(contractTerm);
                    }

                    if (annualContractTerm > 0) {
                        contractTerm_integer = annualContractTerm * 12;
                        sbappend(retstr, each, "~contractTerm_l_c~", string(contractTerm_integer), "|");
                    }
                    //Check the contract month of a SKU and deciding the contract start and end date for the SKU, this changes will play role in ACV TCV also
                    if (contractStartMonth > 0) {
                        contractStartDate_date = strtojavadate(contractStartDate, "MM/dd/yyyy");
                        contractStartDate_date = addmonths(contractStartDate_date, contractStartMonth);
                        contractStartDate = datetostr(contractStartDate_date);
                        if (contractStartMonth >= 12) {
                            sbappend(retstr, each, "~aCVExclusionFlag_l_c~true", "|"); // Exclude From ACV Based On Contract Start Date, this SKU will only participate in TCV
                        }
                    } else {
                        sbappend(retstr, each, "~aCVExclusionFlag_l_c~false", "|"); // Exclude From ACV Based On Contract Start Date, this SKU will only participate in TCV
                    }
                    if (contractEndMonth <= contractTerm_integer) { //if contract term in current quote is higher or equal to the contractEndMonth stored in data table then use below calculation
                        contractStartDate_date = strtojavadate(contractStartDate, "MM/dd/yyyy");
                        contractEndDate_date = addmonths(contractStartDate_date, contractEndMonth);
                        contractEndDate = datetostr(contractEndDate_date);
                        //contractTerm = string(contractEndMonth - contractStartMonth);
                        //sbappend(retstr, each, "~contractTerm_l_c~",contractTerm,"|");
                    }
                    elif(contractEndMonth > contractTerm_integer) { //if contract term in current quote is lesser then the contractEndMonth stored in data table then use below calculation
                        contractStartDate_date = strtojavadate(contractStartDate, "MM/dd/yyyy");
                        contractEndDate_date = addmonths(contractStartDate_date, contractTerm_integer - contractStartMonth);
                        contractEndDate = datetostr(contractEndDate_date);
                        //contractTerm = string(contractTerm_integer - contractStartMonth);
                        //sbappend(retstr, each, "~contractTerm_l_c~",contractTerm,"|");
                    }
                    sbappend(retstr, each, "~contractStartDate_l~", contractStartDate, "|");
                    sbappend(retstr, each, "~contractEndDate_l~", contractEndDate, "|");
                }
                //JIRA 2949 : Start : Build a setup where contract start and end date can be adjusted. There are 2 SKU's, one will charged for first year and second one will be for rest of all the following year.
                GroupRunRate = jsonget(skuNumJson, "GroupRunRate", "float", 0.0);
                GroupFixedMinVal = jsonget(skuNumJson, "GroupFixedMinVal", "float", 0.0);
                GroupMinCalcType = jsonpathgetsingle(pricingJson, "$." + skuNum + ".GroupMinCalcType", "string", ""); //Added by Chandu Pendyala for New Pricing Pattern of RMP 10/13/2025
                dynamicGroupMinimum = jsonpathgetsingle(pricingJson, "$." + skuNum + ".DynamicGroupMinVal", "float", 0.0);
                minContributorSKU = jsonpathgetsingle(pricingJson, "$." + skuNum + ".MinContributorSKU", "string", ""); //Navya: Added this for a new pattern in Payment One
                MinContributionPer = jsonpathgetsingle(pricingJson, "$." + skuNum + ".MinContributionPer", "float", 0.0); //Navya: Added this for a new pattern in Payment One
                aggregateDiscount = jsonpathgetsingle(pricingJson, "$." + skuNum + ".AggregateDiscount", "string", "");
                /*
                    minimumValue = jsonpathgetsingle(pricingJson, "$." + skuNum + ".MinimumValue", "float", 0.0);
                    minimumRunRate = jsonpathgetsingle(pricingJson, "$." + skuNum + ".MinimumRunRate", "float", 0.0);
                */
                minimumValue = jsonget(skuNumJson, "MinimumValue", "float", 0.0);
                minimumRunRate = jsonget(skuNumJson, "MinimumRunRate", "float", 0.0);
                rollupSKU = jsonpathgetsingle(pricingJson, "$." + skuNum + ".RollupSKU", "string", "");
                discountApplicable = jsonpathgetsingle(pricingJson, "$." + skuNum + ".DiscountApplicable", "string", ""); //Is discount applicable on the SKU
                //billingFrequency = jsonpathgetsingle(pricingJson, "$." + skuNum + ".BillingFrequency", "string", "");
                //skuLevelRounding = jsonpathgetsingle(pricingJson, "$." + skuNum + ".Rounding", "string", "");
                skuLevelRounding = jsonpathgetsingle(skuNumJson, "Rounding", "string", "");
                salesUserInput = jsonpathgetsingle(pricingJson, "$." + skuNum + ".salesUserInput", "string", "");
                tCVType = jsonpathgetsingle(pricingJson, "$." + skuNum + ".TCVType", "string", "");
                skipInRenewal = jsonpathgetsingle(pricingJson, "$." + skuNum + ".skipInRenewal", "string", "");
                includeOTFee = jsonpathgetsingle(pricingJson, "$." + skuNum + ".IncludeOTFee", "string", "");
                sKUSpecialTCVInfoJsonArray = jsonpathgetsingle(specialTCVInfoJson, "$." + skuNum, "jsonarray", jsonarray());

                //11/28/2025 - Chandu.P - Updated the key to accomadate the single SKU multiple minimum calculation for Ebank D1B product.
                tieredMinJsonKeyArr = string[];
                append(tieredMinJsonKeyArr, skuNum);
                if (GroupMinCalcType <> "") {
                    append(tieredMinJsonKeyArr, lower(GroupMinCalcType));
                } else {
                    append(tieredMinJsonKeyArr, minimumCalcType);
                }
                tieredMinJsonKey = join(tieredMinJsonKeyArr, "**");
                tieredMinJsonKey = replace(tieredMinJsonKey, " ", "");
                //skuTieredMinJson = jsonget(tieredMinJson, skuNum, "jsonarray", jsonarray());

                skuTieredMinJson = jsonget(tieredMinJson, tieredMinJsonKey, "jsonarray", jsonarray());
                formulaInputSKUs = "";
                adjustedListPrice = 0.0;
                adjustedNetPrice = 0.0;
                adjustedExtendedListPrice = 0.0;
                adjustedExtendedNetPrice = 0.0;
                tierLevelRounding = "";
                discountingNotAvailabel = false;
                calculationExclusionFlag = lower(jsonpathgetsingle(pricingJson, "$." + skuNum + ".ACVTCVExclusion", "string", ""));
                crmACVTCVExclusionFlag = false;
                isUnitRate = false;
                isAggregateHoldingSKU = false;
                //Two Factored Graduted Tier Pattern Changes AMTNB : ATM Content MAnagement
                skuPrimaryAttrFlag = false;
                skuPrimaryAttrQty = 0.0;
                //getting the current(on screen) tier's slabs and their prices to recalculate price of SKU
                if (typeOfPricing == "Aggregate Option Class"
                    OR typeOfPricing == "Asymmetric Tiering Aggregation"
                    OR typeOfPricing == "Average Aggregation Tiering"
                    OR typeOfPricing == "Asymmetric Pricing Tiering Aggregation"
                    OR typeOfPricing == "Aggregate List Price"
                    OR typeOfPricing == "Aggregate Tiers Prices"
                    OR typeOfPricing == "Reduction Aggregation"
                    OR typeOfPricing == "Secondary Tier Aggregation"
                    OR typeOfPricing == "All listPrice TierType Aggregation"
                    OR typeOfPricing == "All TierType Aggregation") { //"Aggregate Tiers Prices" this type of aggregation is speciallly built for Code Connect case
                    jsonput(aggregatePricingJson, skuNum + "**" + modelDocNum, "");
                    aggrSKUJson = jsonget(finalPriceJson, skuNum + "**" + modelDocNum, "json", json());
                    jsonput(aggrSKUJson, "Rounding", skuLevelRounding);
                    jsonput(aggrSKUJson, "TypeOfAggregation", typeOfPricing); //Navya: Added for a new pattern in tokenization
                    if (jsonarraysize(tierDataJsonArr) == 1) {
                        jsonput(aggrSKUJson, "PriceLabel", jsonget(jsonarrayget(tierDataJsonArr, 0, "json"), "PriceLabel", "string", ""));
                    }
                    if (typeOfPricing == "Reduction Aggregation") { //If typeOfPricing == "Reduction Aggregation" then store the extra parameters in FinalPriceJson and use for tier formation.
                        jsonput(aggrSKUJson, "baseMonthlyVolume", jsonget(skuNumJson, "baseMonthlyVolume", "float", 0.0));
                        jsonput(aggrSKUJson, "averageGrowthPerYear", jsonget(skuNumJson, "averageGrowthPerYear", "float", 0.0));
                    }
                    jsonput(finalPriceJson, skuNum + "**" + modelDocNum, aggrSKUJson);
                    crmACVTCVExclusionFlag = true;
                    discountingNotAvailabel = true;
                    isAggregateHoldingSKU = true;
                }
                finalRounding = 10; //Max rounding level assumption
                if (skuLevelRounding <> ""
                    AND isnumber(skuLevelRounding)) {
                    finalRounding = atoi(skuLevelRounding);
                }
                if (minimumCalcType == "group") {
                    //AMTNC -- Norcross model pattern -- Subtotal under subtotal or group minimum under group minimum pattern : Start
                    if (MinimumCalcGrpSKUID <> ""
                        AND MinimumCalcGrpSKUID <> "NA") {
                        jsonput(multiGroupMinJson, MinimumCalcGrpSKUID + "**" + modelDocNum, skuNum + "**" + modelDocNum);
                        isGroupMinimumContributor = true;
                        if (findinarray(jsonkeys(multiGroupMinJson), skuNum + "**" + modelDocNum) == -1) {
                            jsonput(multiGroupMinJson, skuNum + "**" + modelDocNum, "NA");
                        }
                    }
                    //AMTNC -- Norcross model pattern -- Subtotal under subtotal or group minimum under group minimum pattern : Start
                    tempjson = jsonget(groupMinimunJson, skuNum + "**" + modelDocNum, "json", json());
                    jsonput(tempjson, "groupfixedMinimum", GroupFixedMinVal);
                    jsonput(tempjson, "Rounding", finalRounding);
                    jsonput(tempjson, "RunrateRounding", runrateRounding);
                    jsonput(tempjson, "skuTieredMinJson", skuTieredMinJson);
                    jsonput(tempjson, "minTierIdentifier", minTierIdentifier);
                    jsonput(tempjson, "contractTerm", contractTerm);
                    jsonput(tempjson, "realTimeMinTierData", realTimeMinTierData);
                    jsonput(tempjson, "minTierUpdated", minTierUpdated);
                    jsonput(tempjson, "ActionCode", aBOActionCode);
                    jsonput(tempjson, "GroupMinCalcType", GroupMinCalcType); //Added by Chandu Pendyala for New Pricing Pattern of RMP 10/13/2025
                    jsonput(tempjson, "minimumCalcType", minimumCalcType); //Added by Chandu Pendyala for New Pricing Pattern of RMP 10/13/2025
                    jsonput(tempjson, "userInputMinPrice", userInputMinPrice);
                    jsonput(tempjson, "lineLevelTextAreaJson", lineLevelTextAreaJson);
                    jsonput(groupMinimunJson, skuNum + "**" + modelDocNum, tempjson);
                }
                jsonput(lineLevelTextAreaJson, "skuLevelRounding", skuLevelRounding);
                //if action code is Add then only calculate prices
                /* if (aBOActionCode == "ADD"
                    OR aBOActionCode == "RENEW") {
                    
                //updating condition for renewal*/
                if ((aBOActionCode == "ADD"
                        OR aBOActionCode == "RENEW"
                        OR typeOfPricing == "RelationShip Credit")) {
                    if ((aBOActionCode == "RENEW"
                            AND(lower(feeType) == "implementation"
                                OR lower(feeType) == "per occurrence"
                                OR lower(feeType) == "total") AND NOT(renew) AND NOT(jsonget(skuNumJson, "setAboUpdate", "boolean", false))) OR(aBOActionCode <> "ADD"
                            AND skipInRenewal == "true"
                            AND NOT(jsonget(skuNumJson, "bSol_newConfiguration_pf", "boolean", false)))) { //Added feetype = "total" condition by Khasim on 06/02/2026
                        if (includeOTFee <> "true") {
                            sbappend(retstr, each, "~feeType_l_c~", feeType, "|");
                            //sbappend(retstr, each, "~oRCL_ABO_ActionCode_l~NO_UPDATE|");
                            sbappend(retstr, each, "~customActionCode_l_c~NO_UPDATE|");
                            //jsonput(skuNumJson,"ABOActionCode","NO_UPDATE"); // Added by Sateesh 31/7/2025 for Action code double save issue
                            continue; // skip implementation skus that are not selected by user
                        }

                    }
                    elif(aBOActionCode == "RENEW"
                        AND lower(feeType) == "license"
                        AND NOT(jsonget(skuNumJson, "setAboUpdate", "boolean", false))) {
                        //to skip license skus for renewal
                        if (includeOTFee <> "true") {
                            //sbappend(retstr, each, "~oRCL_ABO_ActionCode_l~NO_UPDATE|");
                            sbappend(retstr, each, "~customActionCode_l_c~NO_UPDATE|");
                            //jsonput(skuNumJson,"ABOActionCode","NO_UPDATE"); // Added by Sateesh 31/7/2025 for Action code double save issue
                            continue;
                        }
                    }
                    if (tierUpdated OR tierDiscountApplied) {
                        tierDataJsonArr = jsonget(skuNumJson, "proposalTierDisplay", "jsonarray", jsonarray());
                        if (feeType == "Monthly"
                            OR feeType == "Per Occurrence"
                            OR feeType == "Years") { //Added Per Occurence Feetype, Production defect, JIRA 3349 // Added by Chandu Pendyala for Years Fee type and Billing Frequency 9/23/2025
                            proposalTierDisplayDataForProposal = tierDataJsonArr;
                        }
                        //Getting Last tier row end value of displayed tiers
                        if (isnumber(jsonget(jsonarrayget(tierDataJsonArr, jsonarraysize(tierDataJsonArr) - 1, "json"), "TierEnd", "string", ""))) {
                            nextLowerBound = jsonget(jsonarrayget(tierDataJsonArr, jsonarraysize(tierDataJsonArr) - 1, "json"), "TierEnd", "integer", 0) + 1;
                        } else {
                            nextLowerBound = 10000000;
                        }
                    }
                    //Added By Chirag : 07 May 2025, When line level discount is applied then discounted tiers should be used for calculation
                    /*if(customDiscountType <> "" AND isnumber(customDiscountValue) AND atof(customDiscountValue) > 0.0){
                        tierPriceRecalculationsRequest = json();
                        jsonput(tierPriceRecalculationsRequest, "updatedTierArray", tierDataJsonArr);
                        jsonput(tierPriceRecalculationsRequest, "discountType", customDiscountType);
                        jsonput(tierPriceRecalculationsRequest, "discountValue", atof(customDiscountValue));
                        tierPriceRecalculationsResponse = util.util_tierPriceRecalculations_c(tierPriceRecalculationsRequest);
                        tierDataJsonArr = jsonget(tierPriceRecalculationsResponse, "latestTierArray", "jsonarray", jsonarray());
                        if (feeType == "Monthly") {
                        proposalTierDisplayDataForProposal = tierDataJsonArr;
                        }
                    }*/
                    additionalACV = jsonget(skuNumJson, "AdditionalACV", "float", 0.0);
                    defaultDiscount = jsonget(skuNumJson, "DefaultDiscount", "float", 0.0);
                    // Added : Bibin Sajeevan : 26 Feb 2025 : Adding for Ad Hoc Packageing - Start
                    if (adHocIncludedOrWaivedBool) {
                        /*if(coreProduct == modelVarName){
                            if(adHocCoreType <> "Both"){
                            defaultDiscount = 100.0;
                            }
                            }
                            else{
                            defaultDiscount = 100.0;
                        }*/
                        defaultDiscount = 100.0;
                    }
                    // Added : Bibin Sajeevan : 26 Feb 2025 : Adding for Ad Hoc Packageing - End
                    priceSourceIndicator = "priceList";
                    tierDisplayType = "NA";
                    currentTierIndex = -1;
                    currentTierIndexForMonthlyFee = -1;
                    listPrice = 0.0;
                    listAmount = 0.0;
                    netPrice = 0.0;
                    netAmount = 0.0;
                    roundedListPrice = 0.0;
                    extendedInvoiceMatchRate = 0.0;
                    formattedListPrice = "";
                    formattedNetPrice = "";
                    formattedNetAmount = "";
                    minQuoteNeeded = false;
                    tiersArrSize = jsonarraysize(tierDataJsonArr);
                    isTierEndEqualsMultiplyFactor = false;
                    isSecondLastTier = false;
                    isLastTier = false;
                    remainingPrevQty = previousQtyFactor;
                    previousQtyTierIndex = 0;
                    prevNCurrentQtyFusionBand = 0.0;
                    formulaSKUsetToNoUpdate = false;
                    staticCumGraduatedFinalPriceType = "";
                    additionalDataHeaderFromTierPrice = "";
                    additionalDataFromTierPrice = "";
                    setToNoUpdateAvailable = false;
                    isRoundingDoneInFormula = false; //27 Aug 2025 :- this is specially implemneted for Endpoint Exchange special calculation
                    //In case dynamic unit rate, Recalculating unit rate's to get proper data for aggregation
                    /*if (tiersArrSize > 0 AND aggrRollupSKU <> ""
                    AND aggrRollupSKU <> "NA"
                    AND lower(jsonget(skuFormulaJson, "VariableUnitRate", "string", "")) == "y"
                    AND(tierUpdated == false AND tierDiscountApplied == false)) {*/
                    //if (tiersArrSize > 0 AND lower(jsonget(skuFormulaJson, "VariableUnitRate", "string", "")) == "y" AND(tierUpdated == false AND tierDiscountApplied == false)) {
                    if (tiersArrSize > 0 AND findinarray(variableUnitRateSKUsArr, skuNum) <> -1 AND(tierUpdated == false AND tierDiscountApplied == false)) {
                        //call the util to modify tierdatajsonarr
                        modifyTierDataRequestJson = json();
                        jsonput(modifyTierDataRequestJson, "skuNum", skuNum);
                        jsonput(modifyTierDataRequestJson, "skuLineDataJson", skuNumJson);
                        jsonput(modifyTierDataRequestJson, "TierData", tierDataJsonArr);
                        jsonput(modifyTierDataRequestJson, "pricingJson", pricingJson);
                        jsonput(modifyTierDataRequestJson, "finalPriceJson", finalPriceJson);
                        jsonput(modifyTierDataRequestJson, "modelDocNum", modelDocNum);
                        jsonput(modifyTierDataRequestJson, "variableUnitRateSKUJson", jsonget(variableUnitRateSKUJson, skuNum, "json", json()));
                        modifyTierDataResponseJson = util.util_modifyTierDataJson(modifyTierDataRequestJson);
                        tierDataJsonArr = jsonget(modifyTierDataResponseJson, "TierData", "jsonarray", jsonarray());
                        if (jsonget(modifyTierDataResponseJson, "DynamicMinVal", "float", 0.0) > 0.0) {
                            minimumValue = jsonget(modifyTierDataResponseJson, "DynamicMinVal", "float", 0.0);
                        }
                        if (jsonget(modifyTierDataResponseJson, "setToUpdate", "boolean", false)) { //QTCCPQ-5101 Start
                            sbappend(retstr, each, "~customActionCode_l_c~UPDATE|");
                        }
                        elif(jsonget(modifyTierDataResponseJson, "setToNoUpdate", "boolean", false)) {
                            sbappend(retstr, each, "~customActionCode_l_c~NO_UPDATE|");
                            continue;
                        } //QTCCPQ-5101 End																							     
                    }
                    if ((typeOfTier == "Graduated Tier"
                            OR typeOfTier == "Two Factored Graduated Tier") AND tierDiscountApplied == false AND tierUpdated == false AND tiersArrSize > 0) {
                        tierPriceRecalculationsRequestJson = json();
                        tierArray = jsonput(tierPriceRecalculationsRequestJson, "updatedTierArray", tierDataJsonArr);
                        customDiscountType = jsonput(tierPriceRecalculationsRequestJson, "discountType", customDiscountType);
                        customDiscountVal = jsonput(tierPriceRecalculationsRequestJson, "discountValue", customDiscountValue);
                        tierPriceRecalculationsResponse = util.util_tierPriceRecalculations_c(tierPriceRecalculationsRequestJson);
                        //sbappend(retstr, each, "~lineLevelValidations_l_c~", jsonget(tierPriceRecalculationsResponse, "lineLevelValidations", "string", ""), "|");
                        tierDataJsonArr = jsonget(tierPriceRecalculationsResponse, "latestTierArray", "jsonarray", jsonarray());
                    }
                    //Static Tier Pricing Start
                    // Tier Pricing Start
                    if (typeOfTier == "Static Tier"
                        OR typeOfTier == "Graduated Tier"
                        OR typeOfTier == "Two Factored Graduated Tier"
                        OR typeOfTier == "Overage Tier"
                        OR typeOfTier == "Static Cum Graduated Tier"
                        OR typeOfTier == "Fixed Fee") {

                        reqJson = json();
                        responseJson = json();
                        utilityName = "";

                        // Common request attributes
                        jsonput(reqJson, "typeOfTier", typeOfTier);
                        jsonput(reqJson, "tierDataJsonArr", tierDataJsonArr);
                        jsonput(reqJson, "multiplyFactor", multiplyFactor);
                        jsonput(reqJson, "aggrRollupSKU", aggrRollupSKU);
                        jsonput(reqJson, "priceLabel", priceLabel);
                        jsonput(reqJson, "invoiceMatchRate", invoiceMatchRate);
                        jsonput(reqJson, "userInputPrice", userInputPrice);
                        jsonput(reqJson, "salesUserInput", salesUserInput);
                        jsonput(reqJson, "typeOfPricing", typeOfPricing);
                        jsonput(reqJson, "isUnitRate", isUnitRate);
                        jsonput(reqJson, "modelDocNum", modelDocNum);
                        jsonput(reqJson, "skuNumJson", skuNumJson);
                        jsonput(reqJson, "skuFormulaJson", skuFormulaJson);
                        jsonput(reqJson, "finalPriceJson", finalPriceJson);
                        jsonput(reqJson, "each", each);
                        jsonput(reqJson, "quoteNeeded", quoteNeeded);
                        jsonput(reqJson, "salesQuoteNeeded", salesQuoteNeeded);
                        jsonput(reqJson, "isListPriceNA", isListPriceNA);
                        jsonput(reqJson, "formulaInputSKUs", formulaInputSKUs);
                        jsonput(reqJson, "userInputListPriceCheck", userInputListPriceCheck);
                        jsonput(reqJson, "userTypeInputListPriceCheck", userTypeInputListPriceCheck);
                        jsonput(reqJson, "priceSourceIndicator", priceSourceIndicator);
                        jsonput(reqJson, "setToNoUpdateAvailable", setToNoUpdateAvailable);
                        jsonput(reqJson, "formulaSKUsetToNoUpdate", formulaSKUsetToNoUpdate);
                        jsonput(reqJson, "isTierPricingSKU", isTierPricingSKU);
                        jsonput(reqJson, "isTierEndEqualsMultiplyFactor", isTierEndEqualsMultiplyFactor);
                        jsonput(reqJson, "isSecondLastTier", isSecondLastTier);
                        jsonput(reqJson, "isLastTier", isLastTier);
                        jsonput(reqJson, "tierDiscountApplied", tierDiscountApplied);
                        jsonput(reqJson, "currentTierIndex", currentTierIndex);
                        jsonput(reqJson, "unitRateForAggregate", unitRateForAggregate);
                        jsonput(reqJson, "invoiceMatchRateListPrice", invoiceMatchRateListPrice);
                        jsonput(reqJson, "listPrice", listPrice);
                        jsonput(reqJson, "netPrice", netPrice);
                        jsonput(reqJson, "minimumValue", minimumValue);
                        jsonput(reqJson, "minimumRunRate", minimumRunRate);
                        jsonput(reqJson, "finalRounding", finalRounding);
                        jsonput(reqJson, "isRoundingDoneInFormula", isRoundingDoneInFormula);
                        jsonput(reqJson, "changedQty", changedQty);
                        jsonput(reqJson, "multiParamFormulaJson", multiParamFormulaJson);

                        // Additional attributes needed for graduated/static-cum-graduated flows
                        jsonput(reqJson, "txnType", txnType);
                        jsonput(reqJson, "previousQtyFactor", previousQtyFactor);
                        jsonput(reqJson, "remainingPrevQty", remainingPrevQty);
                        jsonput(reqJson, "skuPrimaryAttrFlag", skuPrimaryAttrFlag);
                        jsonput(reqJson, "skuPrimaryAttrQty", skuPrimaryAttrQty);

                        // Pick utility library based on tier type
                        if (typeOfTier == "Static Tier") {
                            utilityName = "STATIC";
                            responseJson = util.util_staticTierCalculation(reqJson);
                        }
                        elif(typeOfTier == "Overage Tier") {
                            utilityName = "OVERAGE";
                            responseJson = util.util_overageTierCalculation(reqJson);
                        }
                        elif(typeOfTier == "Graduated Tier"
                            OR typeOfTier == "Two Factored Graduated Tier") {
                            utilityName = "GRADUATED";
                            responseJson = util.util_graduatedTierCalculation(reqJson);
                        }
                        elif(typeOfTier == "Static Cum Graduated Tier") {
                            utilityName = "STATIC_CUM_GRADUATED";
                            responseJson = util.util_staticCumGraduatedTierCalculation(reqJson);
                        }
                        elif(typeOfTier == "Fixed Fee") {
                            utilityName = "FIXED_FEE";
                            responseJson = util.util_fixedFeeCalculation(reqJson);
                        }
                        sbappend(retstr, jsonget(responseJson, "returnString", "string", ""));

                        isContinue = jsonget(responseJson, "isContinue", "boolean", false);
                        if (isContinue) {
                            continue;
                        }

                        isListPriceNA = jsonget(responseJson, "isListPriceNA", "boolean", isListPriceNA);
                        listPrice = jsonget(responseJson, "listPrice", "float", listPrice);
                        netPrice = jsonget(responseJson, "netPrice", "float", netPrice);
                        isUnitRate = jsonget(responseJson, "isUnitRate", "boolean", isUnitRate);
                        changedQty = jsonget(responseJson, "changedQty", "float", changedQty);
                        invoiceMatchRateListPrice = jsonget(responseJson, "invoiceMatchRateListPrice", "float", invoiceMatchRateListPrice);
                        formulaInputSKUs = jsonget(responseJson, "formulaInputSKUs", "string", formulaInputSKUs);
                        userInputListPriceCheck = jsonget(responseJson, "userInputListPriceCheck", "boolean", userInputListPriceCheck);
                        userTypeInputListPriceCheck = jsonget(responseJson, "userTypeInputListPriceCheck", "string", userTypeInputListPriceCheck);
                        priceSourceIndicator = jsonget(responseJson, "priceSourceIndicator", "string", priceSourceIndicator);
                        quoteNeeded = jsonget(responseJson, "quoteNeeded", "boolean", quoteNeeded);
                        salesQuoteNeeded = jsonget(responseJson, "salesQuoteNeeded", "boolean", salesQuoteNeeded);
                        priceLabel = jsonget(responseJson, "priceLabel", "string", priceLabel);
                        setToNoUpdateAvailable = jsonget(responseJson, "setToNoUpdateAvailable", "boolean", setToNoUpdateAvailable);
                        formulaSKUsetToNoUpdate = jsonget(responseJson, "formulaSKUsetToNoUpdate", "boolean", formulaSKUsetToNoUpdate);
                        minimumValue = jsonget(responseJson, "minimumValue", "float", minimumValue);
                        minimumRunRate = jsonget(responseJson, "minimumRunRate", "float", minimumRunRate);
                        unitRateForAggregate = jsonget(responseJson, "unitRateForAggregate", "float", unitRateForAggregate);
                        finalRounding = jsonget(responseJson, "finalRounding", "integer", finalRounding);
                        isRoundingDoneInFormula = jsonget(responseJson, "isRoundingDoneInFormula", "boolean", isRoundingDoneInFormula);
                        isTierEndEqualsMultiplyFactor = jsonget(responseJson, "isTierEndEqualsMultiplyFactor", "boolean", isTierEndEqualsMultiplyFactor);
                        isSecondLastTier = jsonget(responseJson, "isSecondLastTier", "boolean", isSecondLastTier);
                        isLastTier = jsonget(responseJson, "isLastTier", "boolean", isLastTier);
                        currentTierIndex = jsonget(responseJson, "currentTierIndex", "integer", currentTierIndex);
                        additionalDataFromTierPrice = jsonget(responseJson, "additionalDataFromTierPrice", "string", additionalDataFromTierPrice);
                        additionalDataHeaderFromTierPrice = jsonget(responseJson, "additionalDataHeaderFromTierPrice", "string", additionalDataHeaderFromTierPrice);
                        isTierPricingSKU = jsonget(responseJson, "isTierPricingSKU", "boolean", isTierPricingSKU);

                        if (utilityName == "GRADUATED"
                            OR utilityName == "OVERAGE") {
                            skuPrimaryAttrFlag = jsonget(responseJson, "skuPrimaryAttrFlag", "boolean", skuPrimaryAttrFlag);
                            skuPrimaryAttrQty = jsonget(responseJson, "skuPrimaryAttrQty", "float", skuPrimaryAttrQty);
                        }

                        if (utilityName == "FIXED_FEE"
                            OR utilityName == "STATIC") {
                            //Included below logic to consider Cost and Margin for Price Calculation of Managed Solution Products - Chandu.P - 3/25/2026
                            cost = jsonget(responseJson, "cost", "float", 0.0);
                            margin = jsonget(responseJson, "margin", "float", 0.0);
                        }

                        if (utilityName == "STATIC_CUM_GRADUATED") {
                            staticCumGraduatedFinalPriceType = jsonget(responseJson, "staticCumGraduatedFinalPriceType", "string", staticCumGraduatedFinalPriceType);
                        }

                        /*if(utilityName == "FIXED_FEE") {
                            formula = jsonget(responseJson, "formula", "string", "");
                        }*/
                    }
                    //Added by Ashok
                    if (listPrice > -1 AND listPrice < 1) {
                        finalRounding = 7;
                    }

                    //Rounding Calculation of listPrice : Start
                    if (NOT isRoundingDoneInFormula) {
                        roundedListPrice = round(listPrice, finalRounding);
                    } else {
                        roundedListPrice = listPrice;
                    }
                    //if (isUnitRate){
                    if (isUnitRate AND NOT(additionalDataFromTierPrice == "Volume"
                            AND additionalDataHeaderFromTierPrice == "Display_Volume")) {
                        //Pranvya's CardSuit case, Price is list Price but still wants show volume on proposal for "Annual Integrator Maintenance Fee"
                        listAmount = listPrice * multiplyFactor;
                        if (invoiceMatchRate > 0) {
                            //extendedInvoiceMatchRate = netPrice * multiplyFactor;
                            extendedInvoiceMatchRate = netPrice;
                        }
                        if (typeOfPricing == "Formula"
                            AND isUnitRate == true AND typeOfTier <> "Fixed Fee") { // Cardsuit Pricing Issue Fix for 15328-17520-69128
                            listAmount = listPrice * changedQty;
                            if (invoiceMatchRate > 0) {
                                extendedInvoiceMatchRate = netPrice * changedQty;
                                //extendedInvoiceMatchRate = netPrice;
                            }
                        }
                    }
                    elif(multiplyFactor == 0) { // When Qty is 0
                        listAmount = listPrice * multiplyFactor;
                        if (invoiceMatchRate > 0) {
                            extendedInvoiceMatchRate = netPrice * multiplyFactor;
                        }
                    }
                    else {
                        listAmount = listPrice;
                    }
                    if (typeOfTier == "Graduated Tier"
                        OR typeOfTier == "Two Factored Graduated Tier") {
                        listAmount = listPrice;
                    }
                    //the below rounding issues done for QTCCPQ-3020 : Start
                    // Added by Ashok Kumar: Feb 10
                    listAmountRounding = 0;
                    if (finalRounding == 0 AND NOT isRoundingDoneInFormula) { //25 Sept 2025 :- JIRA 3135 ,3126 Applied rounding, FIS Teller Elite model ACV TCV and rounding issue
                        listAmountRounding = 0;
                    }
                    elif(listAmount > -1 AND listAmount < 1) {
                        listAmountRounding = 7;
                    }
                    else {
                        listAmountRounding = 2;
                    }
                    listAmount = round(listAmount, listAmountRounding);
                    // End: Ashok Kumar

                    //listAmount = round(listAmount, 2);
                    /* Commented BY Ashok Kumar: Feb 10
                    if (finalRounding == 0 AND NOT isRoundingDoneInFormula) { //25 Sept 2025 :- JIRA 3135 ,3126 Applied rounding, FIS Teller Elite model ACV TCV and rounding issue
                        listAmount = round(listAmount, 0);
                    } else {
                        listAmount = round(listAmount, 2);
                    }
                    */
                    //the below rounding issues done for QTCCPQ-3020 : End
                    //netPrice = listPrice; // did not rounded because We will round it after discount
                    //Rounding Calculation of listPrice : End
                    //Fixed Fee Pricing End
                    if ((calculationExclusionFlag == "y"
                            AND(MinimumCalcGrpSKUID == ""
                                OR MinimumCalcGrpSKUID == "NA") AND(aggrRollupSKU == ""
                                OR aggrRollupSKU == "NA")) OR feeType == "Included") {
                        crmACVTCVExclusionFlag = true;
                    }
                    // Discounting : Start
                    discountPercentage = 0.0;
                    discountActualValue = 0.0;
                    customDiscountVal = 0.0;
                    if (defaultDiscount > 0.0) {
                        customDiscountType = "Percent Off";
                        customDiscountValue = string(defaultDiscount);
                    }
                    if (tierDiscountApplied AND listPrice > 0.0) { // condition updated for renewals
                        discountPercentage = ((listPrice - netPrice) / listPrice) * 100;
                    } else {
                        if (invoiceMatchRateListPrice == 0.0 AND typeOfTier <> "Graduated Tier"
                            AND typeOfTier <> "Two Factored Graduated Tier") {
                            if (isNumber(customDiscountValue)) {
                                customDiscountVal = atof(customDiscountValue);
                            }
                            if (customDiscountType <> "") {
                                if (customDiscountType == "Percent Off") {
                                    discountActualValue = (customDiscountVal * listPrice) / 100;
                                    netPrice = listPrice - discountActualValue;
                                    discountPercentage = customDiscountVal;
                                }
                                elif(customDiscountType == "Amount Off") {
                                    netPrice = listPrice - customDiscountVal;
                                    if (listPrice > 0.0) {
                                        discountPercentage = ((listPrice - netPrice) / listPrice) * 100;
                                    }
                                }
                                elif(customDiscountType == "Price Override") {
                                    if (feeType == "Rebate") {
                                        netPrice = customDiscountVal * -1.0;
                                        if (listPrice > 0.0) {
                                            discountPercentage = ((listPrice - netPrice) / listPrice) * 100;
                                        }
                                    } else {
                                        netPrice = customDiscountVal;
                                        if (listPrice > 0.0) {
                                            discountPercentage = ((listPrice - netPrice) / listPrice) * 100;
                                        }
                                    }
                                }
                                priceSourceIndicator = "manualOverride";
                            }
                        } // Else works for renewal - not updating LIG list price
                        //Sravan : 24March2026 : uncommented elif to fix Retail Origination discount fix
                        elif((typeOfTier == "Graduated Tier"
                                OR typeOfTier == "Two Factored Graduated Tier") AND customDiscountType <> ""
                            AND listPrice <> 0.0) {
                            discountPercentage = ((listPrice - netPrice) / listPrice) * 100;
                            priceSourceIndicator = "manualOverride";
                        }
                        else {
                            if (isNumber(customDiscountValue)) {
                                customDiscountVal = atof(customDiscountValue);
                            }
                            if (customDiscountType <> "") {
                                if (customDiscountType == "Percent Off") {
                                    discountActualValue = (customDiscountVal * invoiceMatchRateListPrice) / 100;
                                    netPrice = invoiceMatchRateListPrice - discountActualValue;
                                    discountPercentage = customDiscountVal;
                                }
                                elif(customDiscountType == "Amount Off") {
                                    netPrice = invoiceMatchRateListPrice - customDiscountVal;
                                    if (invoiceMatchRateListPrice > 0.0) {
                                        discountPercentage = ((invoiceMatchRateListPrice - netPrice) / invoiceMatchRateListPrice) * 100;
                                    }
                                }
                                elif(customDiscountType == "Price Override") {
                                    if (feeType == "Rebate") {
                                        netPrice = customDiscountVal * -1.0;
                                        if (invoiceMatchRateListPrice > 0.0) {
                                            discountPercentage = ((invoiceMatchRateListPrice - netPrice) / invoiceMatchRateListPrice) * 100;
                                        }
                                    } else {
                                        netPrice = customDiscountVal;
                                        if (invoiceMatchRateListPrice > 0.0) {
                                            discountPercentage = ((invoiceMatchRateListPrice - netPrice) / invoiceMatchRateListPrice) * 100;
                                        }
                                    }
                                }
                                priceSourceIndicator = "manualOverride";
                            }
                        }
                    }
                    //Added : Bibin Sajeevan : 13 May 2025 : Setting Dynamic Price Labels
                    if (feeType == "Miscellaneous"
                        AND netPrice == 0 AND priceLabel <> "") {
                        inputJson = json();
                        jsonput(inputJson, "customDiscountType", customDiscountType);
                        jsonput(inputJson, "customDiscountValue", customDiscountValue);
                        jsonput(inputJson, "priceLabel", priceLabel);
                        dynamicPriceLabel = util.util_setMiscellaneousPriceLabel(inputJson);
                    }
                    //Added : Bibin Sajeevan : 13 May 2025 : Setting Dynamic Price Labels
                    jsonput(attributeJson, skuNum, discountPercentage);
                    if (discountPercentage == 100.0) {
                        priceLabel = "Waived";
                    }
                    //Added : Bibin Sajeevan : 26 Feb 2025 : For Ad Hoc Packages
                    if (adHocIncludedOrWaivedBool AND discountPercentage == 100.0 AND(adHocCoreType == "Both"
                            OR adHocCoreType == "Core")) {
                        if (lower(feeType) == "monthly") {
                            priceLabel = "Included";
                        }
                    }
                    //Added : Bibin Sajeevan : 26 Feb 2025 : For Ad Hoc Packages
                    if (defaultDiscount > 0.0) {
                        discountPercentage = 0.0;
                    }
                    // Discounting : End
                    //Rounding Calculation of netPrice & netAmount : Start
                    if (NOT isRoundingDoneInFormula) {
                        netPrice = round(netPrice, finalRounding);
                    }
                    //if (isUnitRate){
                    if (isUnitRate AND NOT(additionalDataFromTierPrice == "Volume"
                            AND additionalDataHeaderFromTierPrice == "Display_Volume")) {
                        //Pranvya's CardSuit case, Price is list Price but still wants show volume on proposal for "Annual Integrator Maintenance Fee"
                        netAmount = netPrice * multiplyFactor;
                        //if (changedQty > 0.0) {
                        if (typeOfPricing == "Formula"
                            AND isUnitRate == true AND typeOfTier <> "Fixed Fee") {
                            netAmount = netPrice * changedQty;
                        }
                    }
                    elif(multiplyFactor == 0) { // When Qty is 0
                        netAmount = netPrice * multiplyFactor;
                    }
                    else {
                        netAmount = netPrice;
                    }
                    if (typeOfTier == "Graduated Tier"
                        OR typeOfTier == "Two Factored Graduated Tier") {
                        netAmount = netPrice;
                    }
                    //netAmount = round(netAmount, 2);
                    netAmountRounding = 0;
                    if (netAmount > -1 AND netAmount < 1) { // Added by Ashok on Feb 10
                        netAmount = round(netAmount, 7);
                        netAmountRounding = 7;
                    } else {
                        netAmount = round(netAmount, 2);
                        netAmountRounding = 2; //// Added by Ashok on Feb 10
                    }

                    //Rounding Calculation of netPrice & netAmount : End
                    //Minimum Calculation : Start
                    minimumRunRateListPrice = (netAmount * minimumRunRate) / 100;
                    if (runrateRounding > 0) {
                        if (minimumRunRateListPrice / runrateRounding > round(minimumRunRateListPrice / runrateRounding, 0)) {
                            minimumRunRateListPrice = (round(minimumRunRateListPrice / runrateRounding, 0) + 1) * runrateRounding;
                        } else {
                            minimumRunRateListPrice = round(minimumRunRateListPrice / runrateRounding, 0) * runrateRounding;
                        }
                    }

                    if (minimumCalcType == lower("Tiered Individual") OR minimumCalcType == lower("TieredRamp Individual") OR minimumCalcType == lower("TieredRampRunrate")) {
                        tempJson = json();
                        jsonput(tempJson, "minimumCalcType", minimumCalcType);
                        jsonput(tempJson, "skuTieredMinJson", skuTieredMinJson);
                        jsonput(tempJson, "MinTierIdentifier", minTierIdentifier);
                        jsonput(tempJson, "contractTerm", contractTerm);
                        jsonput(tempJson, "skuNum", skuNum);
                        jsonput(tempJson, "realTimeMinTierData", realTimeMinTierData);
                        jsonput(tempJson, "minTierUpdated", minTierUpdated);
                        jsonput(tempJson, "minimumRunRateListPrice", minimumRunRateListPrice);
                        jsonput(tempJson, "ActionCode", aBOActionCode);
                        jsonput(tempJson, "listPrice", listPrice);
                        jsonput(tempJson, "netPrice", netPrice);
                        jsonput(tempJson, "listAmount", listAmount);
                        jsonput(tempJson, "netAmount", netAmount);
                        tieredMinimumResponse = util.util_tieredMinimumGenerator(tempJson);
                        minimumValue = jsonget(tieredMinimumResponse, "minValue", "float", 0.0);
                        adjustedListPrice = jsonget(tieredMinimumResponse, "finalAdjustedListPrice", "float", 0.0);
                        adjustedNetPrice = jsonget(tieredMinimumResponse, "finalAdjustedNetPrice", "float", 0.0);
                        adjustedExtendedListPrice = round(jsonget(tieredMinimumResponse, "finalAdjustedExtListPrice", "float", 0.0), 2);
                        adjustedExtendedNetPrice = round(jsonget(tieredMinimumResponse, "finalAdjustedExtNetPrice", "float", 0.0), 2);
                        if (jsonpathcheck(tieredMinimumResponse, "$.minFeeConvertedtoIndividualMin")) {
                            //minFeeConvertedtoIndividualMin = true;
                            minimumCalcType = lower("Individual");
                        }
                        tieredMinimumJsonArray = jsonget(tieredMinimumResponse, "minimumTierDisplay_l_c", "jsonarray", jsonarray());
                        sbappend(retstr, each, "~minimumTierDisplay_l_c~", jsonget(tieredMinimumResponse, "minimumTierDisplay_l_c"), "|");
                    }
                    //if fixed minimum is less the calculated minimum (by run rate) then set the new minimum as calculated minimum (by run rate)
                    /* Commented as part of JIRA QTCCPQ-3901
                    if (minimumValue == -1 AND userInputMinPrice == 0.0) {
                        netPrice = 0.0;
                        netAmount = 0.0;
                        minQuoteNeeded = true;
                    }
                    Commented as part of JIRA QTCCPQ-3901 */
                    //if (minimumValue == -1) { Commented as part of JIRA QTCCPQ-3901
                    if (minimumValue == -1) {
                        displayMinPrice = "Quote";
                        minimumValue = userInputMinPrice;
                        if (minimumValue <= 0) { //added as part of JIRA QTCCPQ-3901
                            //Navya: Below lines were added as part of Defect #976
                            userTypeInputListPriceCheck = "admin_tierMinimum";
                            userInputListPriceCheck = true;
                            netPrice = 0.0;
                            netAmount = 0.0;
                            minQuoteNeeded = true;
                        }
                    }
                    if (minimumRunRateListPrice > minimumValue AND minimumCalcType <> lower("TieredRampRunrate")) {
                        newMinimumVal1 = round(minimumRunRateListPrice, finalRounding);
                    } else {
                        newMinimumVal1 = minimumValue;
                    }
                    minValueBeforeDiscount = newMinimumVal1;

                    /*Apply 100% discount to minimum as well if list price is discounted to 100% : Start*/
                    /*
                        autoMinimumDiscounting = jsonget(lineLevelTextAreaJson, "AutoMinimumDiscounting", "boolean", false);
                        if((discountPercentage < 100.0 AND defaultDiscount < 100.0) AND autoMinimumDiscounting == false AND minimumDiscountType <> "" AND minimumDiscountValue > 0.0){
                        jsonput(lineLevelTextAreaJson, "AutoMinimumDiscounting", false);
                        autoMinimumDiscounting = false;
                        }
                        if((discountPercentage == 100.0 OR defaultDiscount == 100.0) AND minimumDiscountType == "" AND minimumDiscountValue == 0.0){
                        jsonput(lineLevelTextAreaJson, "AutoMinimumDiscounting", false);
                        }
                    autoMinimumDiscounting = jsonget(lineLevelTextAreaJson, "AutoMinimumDiscounting", "boolean", false);*/

                    if (invoiceMatchRateMinimum > 0) {
                        newMinimumVal1 = invoiceMatchRateMinimum;
                    }

                    if (((discountPercentage == 100.0 OR defaultDiscount == 100.0) AND(minimumDiscountType == ""
                            AND minimumDiscountValue == 0.0)) OR adHocIncludedOrWaivedBool) {
                        minimumDiscountType = "Percent Off";
                        minimumDiscountValue = 100.0;
                        jsonput(lineLevelTextAreaJson, "AutoMinimumDiscounting", true);
                        sbappend(retstr, each, "~minimumDiscountType_l_c~", minimumDiscountType, "|");
                        sbappend(retstr, each, "~minimumDiscount_l_c~", string(minimumDiscountValue), "|");
                    }
                    elif(minimumDiscountType <> ""
                        AND jsonget(lineLevelTextAreaJson, "AutoMinimumDiscounting", "boolean", false) == true AND(discountPercentage < 100.0 AND defaultDiscount < 100.0 AND adHocIncludedOrWaivedBool == false)) {
                        minimumDiscountType = "";
                        minimumDiscountValue = 0.0;
                        jsonput(lineLevelTextAreaJson, "AutoMinimumDiscounting", false);
                        sbappend(retstr, each, "~minimumDiscountType_l_c~", "|");
                        sbappend(retstr, each, "~minimumDiscount_l_c~0.0", "|");
                    }
                    /*Apply 100% discount to minimum as well if list price is discounted to 100% : End*/
                    //Discounting on Minimums : Start
                    if (minimumDiscountType <> "") {
                        if (minimumDiscountType == "Percent Off") {
                            minimumDiscountAmount = (minimumDiscountValue * newMinimumVal1) / 100;
                            newMinimumVal1 = newMinimumVal1 - minimumDiscountAmount;
                            minValueDiscountPercent = minimumDiscountValue;
                        }
                        elif(minimumDiscountType == "Amount Off") {
                            if (newMinimumVal1 > 0.0) {
                                minValueDiscountPercent = (minimumDiscountValue / newMinimumVal1) * 100;
                            }
                            newMinimumVal1 = newMinimumVal1 - minimumDiscountValue;
                        }
                        elif(minimumDiscountType == "Price Override") {
                            if (newMinimumVal1 > 0.0) {
                                minValueDiscountPercent = ((newMinimumVal1 - minimumDiscountValue) / newMinimumVal1) * 100;
                            }
                            newMinimumVal1 = minimumDiscountValue;
                        }
                        newMinimumVal1 = round(newMinimumVal1, finalRounding);
                    }
                    //Discounting on Minimums : End

                    //if Extended list price is less than minimum then we are charging the fixed minimum
                    //if (minimumCalcType == "individual" AND newMinimumVal1 > netAmount) {
                    if ((minimumCalcType == lower("Individual") OR minimumCalcType == lower("Tiered Individual") OR minimumCalcType == lower("TieredRamp Individual") OR minimumCalcType == lower("TieredRampRunrate")) AND minQuoteNeeded AND userInputMinPrice == 0.0) {
                        minApplying = true;
                        tempjson = jsonget(finalPriceJson, rollupSKU + "**" + modelDocNum, "json", json());
                        jsonput(tempjson, "IndividualMinApplying", true);
                        jsonput(finalPriceJson, rollupSKU + "**" + modelDocNum, tempJson);
                    }
                    //elif((minimumCalcType == lower("Individual") OR minimumCalcType == lower("Tiered Individual") OR minimumCalcType == lower("TieredRamp Individual") OR minimumCalcType == lower("TieredRampRunrate")) AND newMinimumVal1 > netAmount) {
                    elif((minimumCalcType == lower("Individual") OR minimumCalcType == lower("Tiered Individual") OR minimumCalcType == lower("TieredRamp Individual") OR minimumCalcType == lower("TieredRampRunrate")) AND newMinimumVal1 > netAmount AND NOT((salesQuoteNeeded OR quoteNeeded) AND userInputPrice == 0.0)) {
                        // If a user required price input then do not apply minimum untill user provide price input
                        netAmount = newMinimumVal1;
                        listAmount = newMinimumVal1;
                        minApplying = true;
                        //assumptionDataStr = replace(assumptionDataStr, "@@Monthly Minimum " + skuNum + "@@", formatascurrency(minValue, "USD"));
                        tempjson = jsonget(finalPriceJson, rollupSKU + "**" + modelDocNum, "json", json());
                        jsonput(tempjson, "IndividualMinApplying", true);
                        jsonput(finalPriceJson, rollupSKU + "**" + modelDocNum, tempJson);
                    }
                    minValue = newMinimumVal1;
                    //Minimum Calculation : End
                    //Set the "Needs Product Specialist Input"userInputListPriceCheck
                    if ((needsProductSpecialistInput == false OR miscQuote == false) AND(userTypeInputListPriceCheck == "admin"
                            OR userTypeInputListPriceCheck == "sales") AND userInputPrice == 0.0 /*AND feeType <> "Miscellaneous"*/ ) {
                        if (feeType == "Miscellaneous") {
                            miscQuote = true;
                        } else {
                            needsProductSpecialistInput = true;
                        }
                    }

                    if ((needsProductSpecialistInput == false OR miscQuote == false) AND userInputMinPrice == 0.0 AND displayMinPrice == "Quote" /*AND feeType <> "Miscellaneous"*/ ) {
                        if (feeType == "Miscellaneous") {
                            miscQuote = true;
                        } else {
                            needsProductSpecialistInput = true;
                        }
                    }
                    //Group Minimum Start
                    /*
                    if (MinimumCalcGrpSKUID <> ""
                        AND MinimumCalcGrpSKUID <> "NA") {
                        tempjson = jsonget(finalPriceJson, MinimumCalcGrpSKUID + "**" + modelDocNum, "json", json());
                        grpMinimumDocNum = jsonget(tempjson, "docNum", "string", "");
                        //groupMinSKULineDetail = jsonpathgetsingle(lineDataJson, "$."+grpMinimumDocNum, "json", json());
                        //jsonput(groupMinSKULineDetail, "ABOActionCode", "UPDATE");
                        //jsonput(lineDataJson, grpMinimumDocNum, groupMinSKULineDetail);
                        jsonput(tempjson, "modelVarName", modelVarName);
                        jsonput(tempjson, "aCVTCVPercentContriMonthly", aCVTCVPercentContriMonthly);
                        jsonput(tempjson, "aCVTCVPercentContriImpl", aCVTCVPercentContriImpl);
                        jsonput(tempjson, "aCVTCVPercentContriLicense", aCVTCVPercentContriLicense);
                        if (lower(feeType) == "monthly") {
                            crmACVTCVExclusionFlag = true;
                            if (isUnitRate AND typeOfTier <> "Graduated Tier"
                                AND invoiceMatchRateListPrice > 0.0) {
                                //if (changedQty > 0.0) {
                                if (typeOfPricing == "Formula"
                                    AND isUnitRate == true) {
                                    jsonput(tempjson, "groupListAmount", string((invoiceMatchRateListPrice * changedQty) + jsonget(tempjson, "groupListAmount", "float", 0.0)));
                                } else {
                                    jsonput(tempjson, "groupListAmount", string((invoiceMatchRateListPrice * multiplyFactor) + jsonget(tempjson, "groupListAmount", "float", 0.0)));
                                }
                            }
                            elif(typeOfTier == "Graduated Tier"
                                AND invoiceMatchRateListPrice > 0.0) {
                                jsonput(tempjson, "groupListAmount", string(invoiceMatchRateListPrice + jsonget(tempjson, "groupListAmount", "float", 0.0)));
                            }
                            else {
                                jsonput(tempjson, "groupListAmount", string(listAmount + jsonget(tempjson, "groupListAmount", "float", 0.0)));
                            }
                            //jsonput(tempjson, "groupListAmount", string(listAmount + jsonget(tempjson, "groupListAmount", "float", 0.0)));
                            jsonput(tempjson, "groupNetAmount", string(netAmount + jsonget(tempjson, "groupNetAmount", "float", 0.0)));
                            jsonput(tempjson, "dynamicGroupMinimum", string(dynamicGroupMinimum + jsonget(tempjson, "dynamicGroupMinimum", "float", 0.0)));
                            jsonput(tempjson, "groupRunRateListAmount", string((netAmount * (GroupRunRate / 100)) + jsonget(tempjson, "groupRunRateListAmount", "float", 0.0)));
                            jsonput(tempjson, "groupMinMonthlyProductsSKU", jsonget(tempjson, "groupMinMonthlyProductsSKU", "string", "") + "," + skuNum);
                            if (adHocIncludedOrWaivedBool == false) {
                                jsonput(tempjson, "groupMinadHocIncludedOrWaived", adHocIncludedOrWaivedBool);
                            }
                            isGroupMinimumContributor = true;
                        }
                        elif(lower(feeType) == "implementation"
                            OR lower(feeType) == "per occurrence") {
                            jsonput(tempjson, "groupImplementationListAmount", string(listAmount + jsonget(tempjson, "groupImplementationListAmount", "float", 0.0)));
                            jsonput(tempjson, "groupImplementationNetAmount", string(netAmount + jsonget(tempjson, "groupImplementationNetAmount", "float", 0.0)));
                        }
                        elif(lower(feeType) == "license") {
                            jsonput(tempjson, "groupLicenseListAmount", string(listAmount + jsonget(tempjson, "groupLicenseListAmount", "float", 0.0)));
                            jsonput(tempjson, "groupLicenseNetAmount", string(netAmount + jsonget(tempjson, "groupLicenseNetAmount", "float", 0.0)));
                        }
                        jsonput(tempjson, "groupMinProducts", jsonget(tempjson, "groupMinProducts", "string", "") + "," + skuDesc);
                        jsonput(tempjson, "groupMinProductsSKU", jsonget(tempjson, "groupMinProductsSKU", "string", "") + "," + skuNum);
                        if (findinarray(split(jsonget(tempjson, "groupMinParentSKUs", "string", ""), ","), rollupSKU) == -1) {
                            jsonput(tempjson, "groupMinParentSKUs", jsonget(tempjson, "groupMinParentSKUs", "string", "") + "," + rollupSKU);
                        }
                        jsonput(finalPriceJson, MinimumCalcGrpSKUID + "**" + modelDocNum, tempjson);
                    }*/
                    if (MinimumCalcGrpSKUID <> ""
                        AND MinimumCalcGrpSKUID <> "NA") {
                        grpMinDataCollectorReq = json();
                        jsonput(grpMinDataCollectorReq, "MinimumCalcGrpSKUID", MinimumCalcGrpSKUID);
                        jsonput(grpMinDataCollectorReq, "finalPriceJson", finalPriceJson);
                        jsonput(grpMinDataCollectorReq, "modelDocNum", modelDocNum);
                        jsonput(grpMinDataCollectorReq, "modelVarName", modelVarName);
                        jsonput(grpMinDataCollectorReq, "aCVTCVPercentContriMonthly", aCVTCVPercentContriMonthly);
                        jsonput(grpMinDataCollectorReq, "aCVTCVPercentContriImpl", aCVTCVPercentContriImpl);
                        jsonput(grpMinDataCollectorReq, "aCVTCVPercentContriLicense", aCVTCVPercentContriLicense);
                        jsonput(grpMinDataCollectorReq, "feeType", feeType);
                        jsonput(grpMinDataCollectorReq, "isUnitRate", isUnitRate);
                        jsonput(grpMinDataCollectorReq, "typeOfTier", typeOfTier);
                        jsonput(grpMinDataCollectorReq, "invoiceMatchRateListPrice", invoiceMatchRateListPrice);
                        jsonput(grpMinDataCollectorReq, "changedQty", changedQty);
                        jsonput(grpMinDataCollectorReq, "listAmount", listAmount);
                        jsonput(grpMinDataCollectorReq, "netAmount", netAmount);
                        jsonput(grpMinDataCollectorReq, "dynamicGroupMinimum", dynamicGroupMinimum);
                        jsonput(grpMinDataCollectorReq, "GroupRunRate", GroupRunRate);
                        jsonput(grpMinDataCollectorReq, "skuNum", skuNum);
                        jsonput(grpMinDataCollectorReq, "adHocIncludedOrWaivedBool", adHocIncludedOrWaivedBool);
                        jsonput(grpMinDataCollectorReq, "isGroupMinimumContributor", isGroupMinimumContributor);
                        jsonput(grpMinDataCollectorReq, "rollupSKU", rollupSKU);
                        jsonput(grpMinDataCollectorReq, "skuDesc", skuDesc);
                        // Adding these as they are used in the logic block
                        jsonput(grpMinDataCollectorReq, "typeOfPricing", typeOfPricing);
                        jsonput(grpMinDataCollectorReq, "multiplyFactor", multiplyFactor);
                        jsonput(grpMinDataCollectorReq, "crmACVTCVExclusionFlag", crmACVTCVExclusionFlag);
                        jsonput(grpMinDataCollectorReq, "additionalDataFromTierPrice", additionalDataFromTierPrice);
                        jsonput(grpMinDataCollectorReq, "additionalDataHeaderFromTierPrice", additionalDataHeaderFromTierPrice);
                        jsonput(grpMinDataCollectorReq, "billingFrequency", billingFrequency);
                        jsonput(grpMinDataCollectorReq, "variablepricingSKuJsonforGrpMin", variablepricingSKuJsonforGrpMin); //Added by Basavaraj on 24-MAR-2026 for ARO Bundle SKU's to display unit rate but to include the actual quantity to multiply for group minimum
                        grpMinDataCollectorRes = util.util_groupMinimumDataCollector(grpMinDataCollectorReq);
                        finalPriceJson = jsonget(grpMinDataCollectorRes, "finalPriceJson", "json", finalPriceJson);
                        isGroupMinimumContributor = jsonget(grpMinDataCollectorRes, "isGroupMinimumContributor", "boolean", isGroupMinimumContributor);
                        crmACVTCVExclusionFlag = jsonget(grpMinDataCollectorRes, "crmACVTCVExclusionFlag", "boolean", crmACVTCVExclusionFlag);
                    }
                    //Group Minimum End

                    currentTierIndexForMonthlyFee = currentTierIndex;

                    //Preparing request data and calling util_tierDisplayGenerator to display tier's inside line details : Start
                    if (currentTierIndex > -1 AND NOT(tierUpdated OR tierDiscountApplied)) {
                        currentTierData = jsonarrayget(tierDataJsonArr, currentTierIndex, "json");
                        previousTiers = jsonget(currentTierData, "PreviousTiers", "integer", 0);
                        currentTiers = jsonget(currentTierData, "NextTiers", "integer", 0);
                        tierDisplayType = jsonget(currentTierData, "TierDisplayType", "string", "");
                        if (currentTiers + previousTiers > 1 AND lower(tierDisplayType) <> lower("NA") AND tierDisplayType <> "") {
                            isMultiTierDisplay = true;
                        }
                        tierDisplayPayload = json();
                        jsonput(tierDisplayPayload, "realTimeTierData", realTimeTierData);
                        jsonput(tierDisplayPayload, "TierData", tierDataJsonArr);
                        jsonput(tierDisplayPayload, "CurrentTierIndex", currentTierIndex);
                        jsonput(tierDisplayPayload, "skuNum", skuNum);
                        jsonput(tierDisplayPayload, "skuDesc", skuDesc);
                        jsonput(tierDisplayPayload, "TierDiscApplicable", tierDiscountApplicable);
                        jsonput(tierDisplayPayload, "RequestedValue", multiplyFactor);
                        jsonput(tierDisplayPayload, "TierCalc", tierCalc);
                        jsonput(tierDisplayPayload, "contractStartDate", contractStartDate);
                        jsonput(tierDisplayPayload, "contractEndDate", contractEndDate);
                        jsonput(tierDisplayPayload, "customDiscountValue", customDiscountValue);
                        jsonput(tierDisplayPayload, "customDiscountType", customDiscountType);
                        jsonput(tierDisplayPayload, "modelDocNum", modelDocNum);
                        jsonput(tierDisplayPayload, "aggrRollupSKU", aggrRollupSKU);
                        jsonput(tierDisplayPayload, "isTierEndEqualsMultiplyFactor", isTierEndEqualsMultiplyFactor);
                        jsonput(tierDisplayPayload, "invoiceMatchRate", invoiceMatchRate);
                        jsonput(tierDisplayPayload, "originalTypeofTier", typeOfTier);
                        if (typeOfTier == "Static Tier"
                            OR typeOfTier == "Fixed Fee") {
                            jsonput(tierDisplayPayload, "typeOfTier", "Static");
                        }
                        elif(typeOfTier == "Static Cum Graduated Tier") {
                            jsonput(tierDisplayPayload, "typeOfTier", staticCumGraduatedFinalPriceType);
                        }
                        elif(typeOfTier == "Graduated Tier"
                            OR typeOfTier == "Two Factored Graduated Tier"
                            OR typeOfTier == "Overage Tier") {
                            jsonput(tierDisplayPayload, "typeOfTier", "Graduated");
                        }
                        //Added : Bibin Sajeevan : 26 Feb 2025 : For Ad Hoc Packages
                        if (adHocIncludedOrWaivedBool AND defaultDiscount == 100.0 AND(adHocCoreType == "Both"
                                OR adHocCoreType == "Core")) {
                            if (lower(feeType) == "monthly") {
                                jsonput(tierDisplayPayload, "adHocPriceLabel", "Included");
                            }
                        }
                        //Added : Bibin Sajeevan : 26 Feb 2025 : For Ad Hoc Packages
                        tierGeneratorResponse = util.util_tierDisplayGenerator(tierDisplayPayload);
                        responseJsonArr = jsonget(tierGeneratorResponse, "resultArr", "jsonarray", jsonarray());
                        nextLowerBound = jsonget(tierGeneratorResponse, "lastTierEndValue_Plus_One", "integer", 0);
                        noDiscountJsonArray = jsonget(tierGeneratorResponse, "noDiscountJsonArray", "jsonarray", jsonarray());
                        currentTierIndexForMonthlyFee = jsonget(tierGeneratorResponse, "currentTierIndexForMonthlyFee", "integer", -1);
                        jsonput(lineLevelTextAreaJson, "tierDisplay", noDiscountJsonArray);
                        jsonput(lineLevelTextAreaJson, "showPtdArr", jsonget(tierGeneratorResponse, "priceTypesInSku", "string", ""));
                        jsonput(tierProposalJson, rollupSKU + "**" + modelDocNum, jsonget(tierGeneratorResponse, "tierData", "string", ""));
                        aboTempJson = json();
                        jsonput(aboTempJson, "proposalTierDisplay_l_c", responseJsonArr);
                        proposalTierDisplayDataForProposal = responseJsonArr;
                        jsonput(aboJson, "currentData", aboTempJson);
                        sbappend(retstr, each, "~proposalTierDisplay_l_c~", jsonarraytostr(responseJsonArr), "|");
                        //sbappend(retstr, each, "~lineLevelValidations_l_c~", jsonget(tierGeneratorResponse, "lineLevelValidations", "string", ""), "|");
                    }
                    //Preparing request data and calling util_tierDisplayGenerator to display tier's inside line details : End

                    //Aggregate Tier Pricing Start
                    if (aggrRollupSKU <> ""
                        AND aggrRollupSKU <> "NA") {
                        isAggregatePricingSKU = true;
                        tierFactor1 = 0.0;
                        tierFactor2 = 0.0;
                        aggregationPricingType = jsonpathgetsingle(pricingJson, "$." + aggrRollupSKU + ".TypeOfPricing", "string", "");
                        tempjson = jsonget(finalPriceJson, aggrRollupSKU + "**" + modelDocNum, "json", json());
                        jsonput(tempjson, "modelVarName", modelVarName);
                        jsonput(tempjson, "aCVTCVPercentContriMonthly", aCVTCVPercentContriMonthly);
                        jsonput(tempjson, "aCVTCVPercentContriImpl", aCVTCVPercentContriImpl);
                        jsonput(tempjson, "aCVTCVPercentContriLicense", aCVTCVPercentContriLicense);
                        jsonput(tempjson, "aggregateUnitRate", string(unitRateForAggregate + jsonget(tempjson, "aggregateUnitRate", "float", 0.0)));
                        //QTCCPQ-5239 Added Rebate SKU condition : Start
                        if (jsonget(tempjson, "aggrFeeTypes", "string", "") <> "") {
                            jsonput(tempjson, "aggrFeeTypes", jsonget(tempjson, "aggrFeeTypes", "string", "") + "," + lower(feeType));
                        } else {
                            jsonput(tempjson, "aggrFeeTypes", lower(feeType));
                        }
                        //QTCCPQ-5239 Added Rebate SKU condition : End
                        if (aggregationPricingType == "Reduction Aggregation") {
                            tierFactor1 = jsonget(tempjson, "baseMonthlyVolume", "float", 0.0);
                            tierFactor2 = jsonget(tempjson, "averageGrowthPerYear", "float", 0.0);
                        }
                        //jsonput(tempjson, "aggregatedProducts", jsonget(tempjson, "aggregatedProducts", "string", "") + "," + skuDesc);
                        aggrProdSkuDisplayName = jsonget(tempjson, "aggrProdSkuDisplayName", "string", "");
                        aggrProdSkuNumber = jsonget(tempjson, "aggrProdSkuNumber", "string", "");
                        aggregateImplementationFeeLabel = jsonget(tempjson, "aggregateImplementationFeeLabel", "string", "");
                        aggregateMonthlyFeeLabel = jsonget(tempjson, "aggregateMonthlyFeeLabel", "string", "");
                        aggregateLicenseFeeLabel = jsonget(tempjson, "aggregateLicenseFeeLabel", "string", "");
                        if (rollupSKU <> ""
                            AND rollupSKU <> "NA") {
                            /*
                                aggregatedProductsSKU = jsonget(tempjson, "aggregatedProductsSKU", "string", "");
                                if (find(aggregatedProductsSKU, rollupSKU) == -1) {
                                jsonput(tempjson, "aggregatedProductsSKU", aggregatedProductsSKU + "," + rollupSKU);
                                }
                            */
                            if (findinarray(split(aggrProdSkuNumber, ","), rollupSKU) == -1) {
                                aggrProdSkuNumber = aggrProdSkuNumber + "," + rollupSKU;
                                rollAggrSKUDisplayName = jsonget(tempjson, "skuDisplayName", "string", "");
                                jsonput(tempjson, "aggrProdSkuDisplayName", jsonget(tempjson, "aggrProdSkuDisplayName", "string", "") + "," + rollAggrSKUDisplayName);
                            }
                            if (findinarray(split(aggrProdSkuDisplayName, ","), jsonget(tempjson, "skuDisplayName", "string", "")) == -1) {
                                jsonput(tempjson, "aggrProdSkuDisplayName", aggrProdSkuDisplayName + "," + jsonget(tempjson, "skuDisplayName", "string", ""));
                            }
                        }
                        aggrProdSkuNumber = aggrProdSkuNumber + "," + skuNum;
                        jsonput(tempjson, "aggrProdSkuNumber", aggrProdSkuNumber);
                        if (findinarray(split(aggrProdSkuDisplayName, ","), jsonpathgetsingle(finalPriceJson, "$." + skuNum + "**" + modelDocNum + ".skuDisplayName", "string", "")) == -1) {
                            aggrProdSkuDisplayName = aggrProdSkuDisplayName + "," + jsonpathgetsingle(finalPriceJson, "$." + skuNum + "**" + modelDocNum + ".skuDisplayName", "string", "");
                        }
                        jsonput(tempjson, "aggrProdSkuDisplayName", aggrProdSkuDisplayName);
                        if (lower(feeType) == "implementation"
                            OR lower(feeType) == "per occurrence"
                            OR lower(feeType) == "total") { //Added feetype = "total" condition by Khasim on 06/02/2026
                            jsonput(tempjson, "aggregatedOneTimeProducts", jsonget(tempjson, "aggregatedOneTimeProducts", "string", "") + "," + skuNum);
                            jsonput(tempjson, "aggregateImplementationListPrice", string(listPrice + jsonget(tempjson, "aggregateImplementationListPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                            jsonput(tempjson, "aggregateImplementationListAmount", string(listAmount + jsonget(tempjson, "aggregateImplementationListAmount", "float", 0.0)));
                            jsonput(tempjson, "aggregateImplementationNetPrice", string(netPrice + jsonget(tempjson, "aggregateImplementationNetPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                            jsonput(tempjson, "aggregateImplementationFee", string(netAmount + jsonget(tempjson, "aggregateImplementationFee", "float", 0.0)));
                            if ((quoteNeeded OR salesQuoteNeeded) AND listPrice <= 0.0) {
                                jsonput(tempjson, "aggregateImplementationFeeLabel", "Quote");
                                jsonput(tempjson, "aggregateImplementationListAmount", "0.0");
                                jsonput(tempjson, "aggregateImplementationFee", "0.0");
                            }
                            //Navya: New pattern in tokenization - start
                            if (priceLabel == "Included") {
                                jsonput(tempjson, "aggregateImplementationFeeLabel", "Included");
                                jsonput(tempjson, "aggregateImplementationListAmount", "0.0");
                                jsonput(tempjson, "aggregateImplementationFee", "0.0");
                            }
                            //Navya: New pattern in tokenization - end
                            if (aggregateImplementationFeeLabel == "Quote") {
                                jsonput(tempjson, "aggregateImplementationListAmount", "0.0");
                                jsonput(tempjson, "aggregateImplementationFee", "0.0");
                            }
                        }
                        elif(lower(feeType) == "license") {
                            jsonput(tempjson, "aggregatedOneTimeProducts", jsonget(tempjson, "aggregatedOneTimeProducts", "string", "") + "," + skuNum);
                            jsonput(tempjson, "aggregateLicenseListPrice", string(listPrice + jsonget(tempjson, "aggregateLicenseListPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                            jsonput(tempjson, "aggregateLicenseListAmount", string(listAmount + jsonget(tempjson, "aggregateLicenseListAmount", "float", 0.0)));
                            jsonput(tempjson, "aggregateLicenseNetPrice", string(netPrice + jsonget(tempjson, "aggregateLicenseNetPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                            jsonput(tempjson, "aggregateLicenseFee", string(netAmount + jsonget(tempjson, "aggregateLicenseFee", "float", 0.0)));
                            if ((quoteNeeded OR salesQuoteNeeded) AND listPrice <= 0.0) {
                                jsonput(tempjson, "aggregateLicenseFeeLabel", "Quote");
                                jsonput(tempjson, "aggregateLicenseListAmount", "0.0");
                                jsonput(tempjson, "aggregateLicenseFee", "0.0");
                            }
                            if (aggregateLicenseFeeLabel == "Quote") {
                                jsonput(tempjson, "aggregateLicenseListAmount", "0.0");
                                jsonput(tempjson, "aggregateLicenseFee", "0.0");
                            }

                        }
                        elif(lower(feeType) == "monthly"
                            OR lower(feeType) == "years"
                            OR lower(feeType) == "rebate") {
                            //1. Added by Chandu Pendyala for Years Fee type and Billing Frequency 9/23/2025
                            //2. QTCCPQ-5239 Added Rebate SKU condition
                            aggregateDiscountArr = string[];
                            if (aggregateDiscount <> "NA"
                                AND aggregateDiscount <> "") {
                                aggregateDiscountArr = split(aggregateDiscount, ",");
                            }
                            //aggregationPricingType = jsonpathgetsingle(pricingJson, "$." + aggrRollupSKU + ".TypeOfPricing", "string", "");
                            //Aggregate Tiers Prices : Code Connect changes : Start
                            if (aggregationPricingType == "Aggregate Tiers Prices"
                                AND typeOfTier == "Fixed Fee") {
                                append(aggregateDiscountArr, "100");
                            }
                            //Aggregate Tiers Prices : Code Connect changes : End
                            // Added by Chandu Pendyala to consider Aggregate calculation of SKU's where Fee type and Billing Frequency is Years 9/24/2025
                            if (lower(feeType) == "monthly"
                                OR lower(feeType) == "rebate") { //QTCCPQ-5239 Added Rebate SKU condition
                                if (aggregationPricingType == "Reduction Aggregation") {
                                    jsonput(tempjson, "aggregatedMonthlyProducts", jsonget(tempjson, "aggregatedMonthlyProducts", "string", "") + "," + skuNum);
                                    jsonput(tempjson, "aggregateListPrice", string(listPrice - atof(aggregateDiscountArr[0]) + jsonget(tempjson, "aggregateListPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                                    jsonput(tempjson, "aggregateListAmount", string(listAmount - atof(aggregateDiscountArr[0]) + jsonget(tempjson, "aggregateListAmount", "float", 0.0)));
                                    jsonput(tempjson, "aggregateNetPrice", string(netPrice - atof(aggregateDiscountArr[0]) + jsonget(tempjson, "aggregateNetPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                                    jsonput(tempjson, "aggregateNetAmount", string(netAmount - atof(aggregateDiscountArr[0]) + jsonget(tempjson, "aggregateNetAmount", "float", 0.0)));
                                } else {
                                    jsonput(tempjson, "aggregatedMonthlyProducts", jsonget(tempjson, "aggregatedMonthlyProducts", "string", "") + "," + skuNum);
                                    jsonput(tempjson, "aggregateListPrice", string(listPrice * (atof(aggregateDiscountArr[0]) / 100.0) + jsonget(tempjson, "aggregateListPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                                    jsonput(tempjson, "aggregateListAmount", string(listAmount * (atof(aggregateDiscountArr[0]) / 100.0) + jsonget(tempjson, "aggregateListAmount", "float", 0.0)));
                                    jsonput(tempjson, "aggregateNetPrice", string(netPrice * (atof(aggregateDiscountArr[0]) / 100.0) + jsonget(tempjson, "aggregateNetPrice", "float", 0.0))); //Navya: Added for new pattern in tokenization
                                    jsonput(tempjson, "aggregateNetAmount", string(netAmount * (atof(aggregateDiscountArr[0]) / 100.0) + jsonget(tempjson, "aggregateNetAmount", "float", 0.0)));
                                }
                            }

                            // Added by Chandu Pendyala to consider Aggregate calculation of SKU's where Fee type and Billing Frequency is Years 9/24/2025
                            elif(lower(feeType) == "years") {
                                jsonput(tempjson, "aggregatedAnnualProducts", jsonget(tempjson, "aggregatedAnnualProducts", "string", "") + "," + skuNum);
                                jsonput(tempjson, "aggregateAnnualListPrice", string(listPrice + jsonget(tempjson, "aggregateAnnualListPrice", "float", 0.0)));
                                jsonput(tempjson, "aggregateAnnualListAmount", string(listAmount + jsonget(tempjson, "aggregateAnnualListAmount", "float", 0.0)));
                                jsonput(tempjson, "aggregateAnnualNetPrice", string(netPrice + jsonget(tempjson, "aggregateAnnualNetPrice", "float", 0.0)));
                                jsonput(tempjson, "aggregateAnnualNetAmount", string(netAmount + jsonget(tempjson, "aggregateAnnualNetAmount", "float", 0.0)));
                            }

                            if ((quoteNeeded OR salesQuoteNeeded) AND listPrice <= 0.0) {
                                jsonput(tempjson, "aggregateMonthlyFeeLabel", "Quote");
                                jsonput(tempjson, "aggregateListAmount", "0.0");
                                jsonput(tempjson, "aggregateNetAmount", "0.0");
                            }
                            if (aggregateMonthlyFeeLabel == "Quote") {
                                jsonput(tempjson, "aggregateListAmount", "0.0");
                                jsonput(tempjson, "aggregateNetAmount", "0.0");
                            }
                            //Added by Aditya for QTCCPQ-4550 - START
                            if (priceLabel == "Included") {
                                existingLabel = jsonget(tempjson, "aggregateMonthlyFeeLabel", "string", "");
                            if (existingLabel == "" OR existingLabel == "Included") {
                                jsonput(tempjson, "aggregateMonthlyFeeLabel", "Included");
                            }
                        }
                            if (priceLabel == "Waived") {
                                jsonput(tempjson, "aggregateMonthlyFeeLabel", "");
                            }
                            //Added by Aditya for QTCCPQ-4550 - END
                            if (MinimumCalcGrpSKUID <> ""
                                AND MinimumCalcGrpSKUID <> "NA") {
                                //Monthly SKU's which have both group min and aggregate
                                jsonput(tempjson, "aggregatePlusMinListAmount", string(listAmount + jsonget(tempjson, "aggregatePlusMinListAmount", "float", 0.0)));
                                jsonput(tempjson, "aggregatePlusMinNetAmount", string(netAmount + jsonget(tempjson, "aggregatePlusMinNetAmount", "float", 0.0)));
                            }
                            jsonput(tempjson, "uOM", uOM);
                            nextTierUnitRates = jsonget(tempjson, "nextTierUnitRates", "string", "");
                            aggregateTierStart = jsonget(tempjson, "aggregateTierStart", "string", "");
                            aggregateTierEnd = jsonget(tempjson, "aggregateTierEnd", "string", "");
                            tiersListPrices = jsonget(tempjson, "tiersListPrices", "string", "");
                            if (currentTierIndex > 0) {
                                counter = -1;
                            } else {
                                counter = 0;
                            }
                            if (aggregationPricingType == "All TierType Aggregation") {
                                currentTierIndex = 0;
                                counter = 0;
                            }
                            tempindex = 0;
                            tempArr = string[];
                            listTempArr = string[];
                            aggregateTierStartArr = string[];
                            aggregateTierEndArr = string[];
                            if (nextTierUnitRates <> "") {
                                tempArr = split(nextTierUnitRates, ",");
                            }
                            if (tiersListPrices <> "") {
                                listTempArr = split(tiersListPrices, ",");
                            }
                            if (aggregateTierStart <> "") {
                                aggregateTierStartArr = split(aggregateTierStart, ",");
                            }
                            if (aggregateTierEnd <> "") {
                                aggregateTierEndArr = split(aggregateTierEnd, ",");
                            }
                            indexArr = range(tiersArrSize);
                            numOfAggTiers = 0;
                            if (aggregationPricingType == "All listPrice TierType Aggregation") {
                                if (jsonarraysize(proposalTierDisplayDataForProposal) < sizeofarray(aggregateDiscountArr)) {
                                    remove(aggregateDiscountArr, sizeofarray(aggregateDiscountArr) - 1);
                                }
                            }
                            sizeofAggrDiscArray = sizeofarray(aggregateDiscountArr);
                            atpTierIndex = 0;
                            for i in aggregateDiscountArr {
                                //if ((aggregationPricingType == "Aggregate Tiers Prices" AND jsonarraysize(tierDataJsonArr) >= sizeofAggrDiscArray) OR(aggregationPricingType == "All listPrice TierType Aggregation" AND jsonarraysize(proposalTierDisplayDataForProposal) >= sizeofAggrDiscArray)) {
                                if ((aggregationPricingType == "Aggregate Tiers Prices"
                                        AND jsonarraysize(tierDataJsonArr) >= sizeofAggrDiscArray) OR(aggregationPricingType == "All listPrice TierType Aggregation"
                                        AND atpTierIndex < jsonarraysize(proposalTierDisplayDataForProposal))) {
                                    atpTierJson = jsonarrayget(tierDataJsonArr, atpTierIndex, "json");
                                    tierStartName = "TierStart";
                                    tierEndName = "TierEnd";
                                    if (aggregationPricingType == "All listPrice TierType Aggregation"
                                        AND NOT(tierUpdated OR tierDiscountApplied)) {
                                        atpTierJson = jsonarrayget(proposalTierDisplayDataForProposal, atpTierIndex, "json");
                                        tierStartName = "ptdArr_tierStart_l_c";
                                        tierEndName = "ptdArr_tierEnd_l_c";
                                    }
                                    if (sizeofarray(tempArr) > 0 AND isnumber(tempArr[tempindex])) {
                                        tempArr[tempindex] = string(atof(tempArr[tempindex]) + jsonget(atpTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                        listTempArr[tempindex] = string(atof(listTempArr[tempindex]) + jsonget(atpTierJson, "ptdArr_discountedListPrice_l_c", "float", 0.0));
                                        if (typeOfTier <> "Fixed Fee") {
                                            aggregateTierStartArr[tempindex] = string(integer(jsonget(atpTierJson, tierStartName, "float", 0.0)));
                                            if (sizeofAggrDiscArray - 1 == atpTierIndex AND aggregationPricingType <> "All listPrice TierType Aggregation") {
                                                aggregateTierEndArr[tempindex] = "Over";
                                            } else {
                                                if (jsonget(atpTierJson, tierEndName, "string", "") == "Over") {
                                                    aggregateTierEndArr[tempindex] = "Over";
                                                } else {
                                                    aggregateTierEndArr[tempindex] = string(integer(jsonget(atpTierJson, tierEndName, "float", 0.0)));
                                                }
                                            }
                                        }
                                    } else {
                                        tempArr[tempindex] = string(jsonget(atpTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                        listTempArr[tempindex] = string(jsonget(atpTierJson, "ptdArr_discountedListPrice_l_c", "float", 0.0));
                                        if (typeOfTier <> "Fixed Fee") {
                                            aggregateTierStartArr[tempindex] = string(integer(jsonget(atpTierJson, tierStartName, "float", 0.0)));
                                            if (sizeofAggrDiscArray - 1 == atpTierIndex AND aggregationPricingType <> "All listPrice TierType Aggregation") {
                                                aggregateTierEndArr[tempindex] = "Over";
                                            } else {
                                                if (jsonget(atpTierJson, tierEndName, "string", "") == "Over") {
                                                    aggregateTierEndArr[tempindex] = "Over";
                                                } else {
                                                    aggregateTierEndArr[tempindex] = string(integer(jsonget(atpTierJson, tierEndName, "float", 0.0)));
                                                }
                                            }
                                        }
                                    }
                                    atpTierIndex = atpTierIndex + 1;
                                    tempindex = tempindex + 1;
                                    //numOfAggTiers = numOfAggTiers + 1;
                                } else {
                                    if (findinarray(indexArr, currentTierIndex + counter) <> -1) {
                                        nextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex + counter, "json");
                                        if (aggregationPricingType == "Reduction Aggregation"
                                            OR aggregationPricingType == "Secondary Tier Aggregation") {
                                            nextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex, "json");
                                        }
                                        if (jsonarraysize(tierDataJsonArr) > (currentTierIndex + counter + 1)) {
                                            nextTierEndString = jsonget(jsonarrayget(tierDataJsonArr, currentTierIndex + counter + 1, "json"), "TierEnd", "string", "");
                                            if (lower(nextTierEndString) == "over") {
                                                nextTierEnd = 9999999999.0;
                                            } else {
                                                nextTierEnd = atof(nextTierEndString);
                                            }
                                        } else {
                                            nextTierEnd = 9999999999.0;
                                        }
                                        //aggregationPricingType = jsonpathgetsingle(pricingJson, "$." + aggrRollupSKU + ".TypeOfPricing", "string", "");
                                        //Jira - 3604 start : If tier discount type is Price Override, the entered amount should be as-is and must not apply any aggregate discount on top of it.
                                        if (jsonget(nextTierJson, "ptdArr_discountType_l_c", "string", "") == "Price Override"
                                            AND tempindex <> 0) {
                                            i = "0";
                                        }
                                        //Jira-3604 end
                                        if (aggregationPricingType == "Reduction Aggregation") {
                                            discountVal = atof(i);
                                        } else {
                                            discountVal = 1 - (atof(i) / 100);
                                        }
                                        if (tempindex == 0) {
                                            aggregateTierStartArr[tempindex] = "0";
                                            if (currentTierIndex == 0 AND aggregationPricingType == "Asymmetric Tiering Aggregation") {
                                                aggregateTierEndArr[tempindex] = string(integer(jsonget(nextTierJson, "TierEnd", "float", 0.0)));
                                            }
                                            elif(aggregationPricingType == "Reduction Aggregation") {
                                                aggregateTierEndArr[tempindex] = string(integer(tierFactor1));
                                            }
                                            elif(aggregationPricingType == "Secondary Tier Aggregation") {
                                                aggregateTierEndArr[tempindex] = additionalDataFromTierPrice;
                                            }
                                            elif(aggregationPricingType == "All TierType Aggregation") {
                                                aggregateTierEndArr[tempindex] = "Over";
                                            }
                                            elif(typeOfTier == "Overage Tier") {
                                                overageTierEnd = jsonget(jsonarrayget(proposalTierDisplayDataForProposal, 0, "json"), "ptdArr_tierEnd_l_c", "float", 0.0);
                                                if (multiplyFactor >= overageTierEnd) {
                                                    aggregateTierEndArr[tempindex] = string(integer(multiplyFactor));
                                                } else {
                                                    aggregateTierEndArr[tempindex] = string(integer(overageTierEnd));
                                                }
                                            }
                                            else {
                                                aggregateTierEndArr[tempindex] = string(integer(multiplyFactor));
                                            }
                                        }
                                        elif(tempindex == 1 AND aggregationPricingType <> "All TierType Aggregation") {
                                            if (currentTierIndex == 0 AND aggregationPricingType == "Asymmetric Tiering Aggregation") {
                                                asymmetricNextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex + counter + 1, "json");
                                                aggregateTierStartArr[tempindex] = string(integer(jsonget(asymmetricNextTierJson, "TierStart", "float", 0.0)));
                                            }
                                            elif(aggregationPricingType == "Reduction Aggregation") {
                                                aggregateTierStartArr[tempindex] = string(integer(tierFactor1 + 1));
                                            }
                                            elif(aggregationPricingType == "Secondary Tier Aggregation") {
                                                aggregateTierStartArr[tempindex] = string(atoi(additionalDataFromTierPrice) + 1);
                                            }
                                            elif(typeOfTier == "Overage Tier") {
                                                aggregateTierStartArr[tempindex] = string(atoi(aggregateTierEndArr[tempindex - 1]) + 1);
                                            }
                                            else {
                                                aggregateTierStartArr[tempindex] = string(integer(multiplyFactor + 1));
                                            }
                                            if (aggregationPricingType == "Reduction Aggregation") {
                                                if ((isSecondLastTier AND isTierEndEqualsMultiplyFactor) OR sizeofAggrDiscArray - 1 == tempindex OR isLastTier) {
                                                    aggregateTierEndArr[tempindex] = "Over";
                                                }
                                                elif((tierFactor1 + tierFactor2) % 1000 > 0) {
                                                    aggregateTierEndArr[tempindex] = string(integer((tierFactor1 + tierFactor2) + 1000 - (tierFactor1 + tierFactor2) % 1000));
                                                }
                                                else {
                                                    aggregateTierEndArr[tempindex] = string(integer(tierFactor1 + tierFactor2));
                                                }
                                            }
                                            elif(isnumber(jsonget(nextTierJson, "TierEnd", "string", ""))) {
                                                if ((isSecondLastTier AND isTierEndEqualsMultiplyFactor) OR isLastTier OR(sizeofAggrDiscArray - 1 == tempindex)) {
                                                    aggregateTierEndArr[tempindex] = "Over";
                                                } else {
                                                    aggregateTierEndArr[tempindex] = string(integer(jsonget(nextTierJson, "TierEnd", "float", 0.0)));
                                                }
                                            }
                                            else {
                                                aggregateTierEndArr[tempindex] = jsonget(nextTierJson, "TierEnd", "string", "");
                                            }
                                        }
                                        elif(tempindex + 1 < sizeofAggrDiscArray AND aggregationPricingType <> "All TierType Aggregation") {
                                            if (aggregationPricingType == "Reduction Aggregation") {
                                                aggregateTierStartArr[tempindex] = string(atoi(aggregateTierEndArr[tempindex - 1]) + 1);
                                                if ((atoi(aggregateTierEndArr[tempindex - 1]) + tierFactor2) % 1000 > 0) {
                                                    aggregateTierEndArr[tempindex] = string(integer((atoi(aggregateTierEndArr[tempindex - 1]) + tierFactor2) + 1000 - (atoi(aggregateTierEndArr[tempindex - 1]) + tierFactor2) % 1000));
                                                } else {
                                                    aggregateTierEndArr[tempindex] = string(integer(atoi(aggregateTierEndArr[tempindex - 1]) + tierFactor2));
                                                }
                                            } else {
                                                aggregateTierStartArr[tempindex] = string(integer(jsonget(nextTierJson, "TierStart", "float", 0.0)));
                                                aggregateTierEndArr[tempindex] = jsonget(nextTierJson, "TierEnd", "string", "");
                                            }
                                        }
                                        elif(aggregationPricingType <> "All TierType Aggregation") {
                                            if (aggregationPricingType == "Reduction Aggregation") {
                                                aggregateTierStartArr[tempindex] = string(atoi(aggregateTierEndArr[tempindex - 1]) + 1);
                                            } else {
                                                aggregateTierStartArr[tempindex] = string(integer(jsonget(nextTierJson, "TierStart", "float", 0.0)));
                                            }
                                            aggregateTierEndArr[tempindex] = "Over";
                                        }

                                        if (tempindex == 0) {
                                            aggregateNetAmtKey = "aggregateNetAmount";
                                            //Updated below logic to consider Years Fee type while calculating Aggregation Net Amount - Chandu.P - 3/30/2026
                                            if (lower(feeType) == "years") {
                                                aggregateNetAmtKey = "aggregateAnnualNetAmount";
                                            }


                                            if (aggregationPricingType == "Reduction Aggregation") {
                                                tempArr[tempindex] = string(jsonget(tempjson, aggregateNetAmtKey, "float", 0.0) * (1 - discountVal));
                                            } else {
                                                tempArr[tempindex] = string(jsonget(tempjson, aggregateNetAmtKey, "float", 0.0) * (atof(i) / 100));
                                            }

                                        }
                                        elif(isnumber(tempArr[tempindex]) AND aggregationPricingType <> "All TierType Aggregation") {
                                            // Sandeep: Added New Pricing Scenario "Average Aggregation Tiering" : Start
                                            if (aggregationPricingType == "Average Aggregation Tiering") {
                                                if (multiplyFactor > 0) {
                                                    tempArr[tempindex] = string(atof(tempArr[tempindex]) + netAmount / multiplyFactor);
                                                } else {
                                                    tempArr[tempindex] = string(atof(tempArr[tempindex]));
                                                }
                                            }
                                            // Sandeep: Added New Pricing Scenario "Average Aggregation Tiering" : End
                                            elif(currentTierIndex == 0 AND aggregationPricingType == "Asymmetric Tiering Aggregation") {
                                                asymmetricNextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex + counter + 1, "json");
                                                tempArr[tempindex] = string(atof(tempArr[tempindex]) + (discountVal * jsonget(asymmetricNextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0)));
                                            }
                                            elif(currentTierIndex == 0 AND aggregationPricingType == "Asymmetric Pricing Tiering Aggregation") {
                                                asymmetricNextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex + counter + 1, "json");
                                                tempArr[tempindex] = string(atof(tempArr[tempindex]) + (discountVal * jsonget(asymmetricNextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0)));
                                            }
                                            elif(aggregationPricingType == "Reduction Aggregation") {
                                                tempArr[tempindex] = string(atof(tempArr[tempindex]) + (jsonget(nextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0) - discountVal));
                                            }
                                            elif(typeOfTier == "Overage Tier"
                                                AND tempindex > 0) {
                                                tempArr[tempindex] = string(atof(tempArr[tempindex]) + jsonget(jsonarrayget(proposalTierDisplayDataForProposal, tempindex, "json"), "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                            }
                                            else {
                                                tempArr[tempindex] = string(atof(tempArr[tempindex]) + (discountVal * jsonget(nextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0)));
                                            }
                                        }
                                        elif(aggregationPricingType <> "All TierType Aggregation") {
                                            // Sandeep: Added New Pricing Scenario "Average Aggregation Tiering" : Start
                                            if (aggregationPricingType == "Average Aggregation Tiering") {
                                                if (multiplyFactor > 0) {
                                                    tempArr[tempindex] = string(netAmount / multiplyFactor);
                                                } else {
                                                    tempArr[tempindex] = string(0.0);
                                                }
                                            }
                                            // Sandeep: Added New Pricing Scenario "Average Aggregation Tiering" : End
                                            elif(currentTierIndex == 0 AND aggregationPricingType == "Asymmetric Tiering Aggregation") {
                                                asymmetricNextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex + counter + 1, "json");
                                                tempArr[tempindex] = string(discountVal * jsonget(asymmetricNextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                            }
                                            elif(currentTierIndex == 0 AND aggregationPricingType == "Asymmetric Pricing Tiering Aggregation") {
                                                asymmetricNextTierJson = jsonarrayget(tierDataJsonArr, currentTierIndex + counter + 1, "json");
                                                tempArr[tempindex] = string(discountVal * jsonget(asymmetricNextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                            }
                                            elif(aggregationPricingType == "Reduction Aggregation") {
                                                tempArr[tempindex] = string(jsonget(nextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0) - discountVal);
                                            }
                                            elif(typeOfTier == "Overage Tier"
                                                AND tempindex > 0) {
                                                tempArr[tempindex] = string(jsonget(jsonarrayget(proposalTierDisplayDataForProposal, tempindex, "json"), "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                            }
                                            else {
                                                tempArr[tempindex] = string(discountVal * jsonget(nextTierJson, "ptdArr_discountedUnitRate_l_c", "float", 0.0));
                                            }
                                        }
                                        aggrTierEnd = 0.0;
                                        if (isnumber(jsonget(nextTierJson, "TierEnd", "string", ""))) {
                                            aggrTierEnd = jsonget(nextTierJson, "TierEnd", "float", 0.0);
                                        } else {
                                            aggrTierEnd = 9999999999.0;
                                        }
                                        if (counter == 0 AND numOfAggTiers == 0 AND jsonget(nextTierJson, "TierEnd", "float", 0.0) <> multiplyFactor) {
                                            counter = counter;
                                        }
                                        elif(counter == -1 AND numOfAggTiers == 0 AND nextTierEnd == multiplyFactor) {
                                            counter = counter + 2;
                                        }
                                        else {
                                            counter = counter + 1;
                                        }
                                        numOfAggTiers = numOfAggTiers + 1;
                                    } else {
                                        break;
                                    }
                                    tempindex = tempindex + 1;
                                }
                            }
                            jsonput(tempJson, "staticCumGraduatedFinalPriceType", staticCumGraduatedFinalPriceType);
                            jsonput(tempJson, "tiersListPrices", join(listTempArr, ","));
                            jsonput(tempjson, "nextTierUnitRates", join(tempArr, ","));
                            jsonput(tempjson, "aggregateTierStart", join(aggregateTierStartArr, ","));
                            jsonput(tempjson, "aggregateTierEnd", join(aggregateTierEndArr, ","));
                            jsonput(tempjson, "numberOfAggTiers", numOfAggTiers);
                        }
                        jsonput(finalPriceJson, aggrRollupSKU + "**" + modelDocNum, tempjson);
                    }
                    //Aggregate Tier Pricing End
                    jsonput(skuNumJson, "ListPrice", listPrice);
                    jsonput(skuNumJson, "ExtendedListPrice", listAmount);
                    jsonput(skuNumJson, "finalRounding", finalRounding);
                    jsonput(lineDataJson, each, skuNumJson);
                    //Start : populating basic line level attriute's
                    if (typeOfPricing <> "Aggregate Option Class"
                        AND typeOfPricing <> "Asymmetric Tiering Aggregation"
                        AND typeOfPricing <> "Option Class"
                        AND typeOfPricing <> "Asymmetric Pricing Tiering Aggregation"
                        AND typeOfPricing <> "Average Aggregation Tiering"
                        AND typeOfPricing <> "Aggregate List Price"
                        AND typeOfPricing <> "Aggregate Tiers Prices"
                        AND typeOfPricing <> "Reduction Aggregation"
                        AND typeOfPricing <> "Secondary Tier Aggregation"
                        AND typeOfPricing <> "All listPrice TierType Aggregation"
                        AND typeOfPricing <> "All TierType Aggregation") {
                        sbappend(retstr, each, "~priceSourceIndicator_l_c~", priceSourceIndicator, "|");
                    }
                    sbappend(retstr, each, "~userInputListPrice_l_c~", string(userInputListPriceCheck), "|"); //Flag For user input SKU's
                    sbappend(retstr, each, "~userTypeInputPriceRequired_l_c~", userTypeInputListPriceCheck, "|"); //Flag For which usertype price input is required
                    sbappend(retstr, each, "~listPrice_l~", string(listPrice), "|");
                    //formatting of displaylistprice and displaynetprice : Start
                    if (typeOfPricing <> "Aggregate Option Class"
                        AND typeOfPricing <> "Asymmetric Tiering Aggregation"
                        AND typeOfPricing <> "Asymmetric Pricing Tiering Aggregation"
                        AND typeOfPricing <> "Average Aggregation Tiering"
                        AND typeOfPricing <> "Aggregate List Price"
                        AND typeOfPricing <> "Aggregate Tiers Prices"
                        AND typeOfPricing <> "Reduction Aggregation"
                        AND typeOfPricing <> "Secondary Tier Aggregation"
                        AND typeOfPricing <> "All listPrice TierType Aggregation"
                        AND typeOfPricing <> "All TierType Aggregation") {
                        formattedListPrice = util.util_formatCurrency(roundedListPrice, "USD", finalRounding);
                        formattedNetPrice = util.util_formatCurrency(netPrice, "USD", finalRounding);
                        //formattedNetAmount = util.util_formatCurrency(netAmount, "USD", 2);
                        formattedNetAmount = util.util_formatCurrency(netAmount, "USD", netAmountRounding); // Added by Ashok on Feb 10
                    }
                    //formatting of displaylistprice and displaynetprice : End
                    if ((adHocIncludedOrWaivedBool OR(adHocIncludedOrWaivedBool AND discountPercentage == 100.0 AND(adHocCoreType == "Both"
                            OR adHocCoreType == "Core"))) AND listPrice > 0.0) {
                        sbappend(retstr, each, "~displayListPrice_l_c~", formattedListPrice, "|");
                    }
                    elif(priceLabel <> ""
                        AND priceLabel <> "Waived") {
                        sbappend(retstr, each, "~displayListPrice_l_c~", priceLabel, "|");
                    }
                    elif(isListPriceNA) {
                        sbappend(retstr, each, "~displayListPrice_l_c~n/a|");
                    }
                    elif(quoteNeeded) {
                        sbappend(retstr, each, "~displayListPrice_l_c~Quote|");
                    }
                    elif(salesQuoteNeeded) {
                        sbappend(retstr, each, "~displayListPrice_l_c~Quote|");
                    }
                    /*
                        elif(minQuoteNeeded){
                        sbappend(retstr, each, "~displayListPrice_l_c~Quote|");
                        }
                    */
                    elif(feeType == "Included") {
                        sbappend(retstr, each, "~displayListPrice_l_c~Included|");
                    }
                    elif(feeType == "Pass-Thru") {
                        sbappend(retstr, each, "~displayListPrice_l_c~Pass-Thru|");
                    }
                    else {
                        sbappend(retstr, each, "~displayListPrice_l_c~", formattedListPrice, "|");
                    }
                    sbappend(retstr, each, "~netPrice_l~", string(netPrice), "|");
                    /*if (defaultDiscount == 100.0){
                        priceLabel = "Waived";
                        }*/
                    //Added : Bibin Sajeevan : 13 May 2025 : Dynamic Price Label for Miscellaneous Fees
                    if (dynamicPriceLabel <> ""
                        AND priceLabel <> "Waived") {
                        sbappend(retstr, each, "~displayNetPrice_l_c~", dynamicPriceLabel, "|");
                    }
                    //Added : Bibin Sajeevan : 13 May 2025 : Dynamic Price Label for Miscellaneous Fees
                    //if (priceLabel <> "") {
                    elif(priceLabel <> "") {
                        sbappend(retstr, each, "~displayNetPrice_l_c~", priceLabel, "|");
                    }
                    elif(isListPriceNA) {
                        sbappend(retstr, each, "~displayNetPrice_l_c~n/a|");
                    }
                    elif(quoteNeeded) {
                        sbappend(retstr, each, "~displayNetPrice_l_c~Quote|");
                    }
                    elif(salesQuoteNeeded) {
                        sbappend(retstr, each, "~displayNetPrice_l_c~Quote|");
                    }
                    elif(minQuoteNeeded) {
                        sbappend(retstr, each, "~displayNetPrice_l_c~Quote|");
                    }
                    elif(feeType == "Included") {
                        sbappend(retstr, each, "~displayNetPrice_l_c~Included|");
                    }
                    elif(feeType == "Pass-Thru") {
                        sbappend(retstr, each, "~displayNetPrice_l_c~Pass-Through|");
                    }
                    else {
                        sbappend(retstr, each, "~displayNetPrice_l_c~", formattedNetPrice, "|");
                    }
                    /*Introducing/Setting String attribute of extendend net price : Start*/
                    if (priceLabel == "Waived") {
                        sbappend(retstr, each, "~extendedProposedPrice_l_c~Waived|");
                    } else {
                        sbappend(retstr, each, "~extendedProposedPrice_l_c~", formattedNetAmount, "|");
                    }
                    /*Introducing/Setting String attribute of extendend net price : Start*/
                    //Added by Sateesh Somisetti 18th July

                    PreviousListPrice = jsonget(skuNumJson, "PreviousListPrice", "float", 0.0);
                    PreviousNetPrice = jsonget(skuNumJson, "PreviousNetPrice", "float", 0.0);
                    PreviousListAmount = jsonget(skuNumJson, "PreviousListAmount", "float", 0.0);
                    //PreviousNetAmount = jsonget(skuNumJson, "PreviousNetAmount", "float", 0.0);
                    PreviousNetAmount = PreviousNetPrice;
                    if (NOT(minimumCalcType == lower("TieredRamp Individual") OR minimumCalcType == lower("TieredRampRunrate"))) {
                        if (jsonpathcheck(skuNumJson, "$.PreviousListPrice") AND jsonpathcheck(skuNumJson, "$.PreviousNetPrice") AND jsonpathcheck(skuNumJson, "$.PreviousListAmount") AND jsonpathcheck(skuNumJson, "$.PreviousNetAmount") AND NOT(isnull(PreviousListPrice)) AND NOT(isnull(PreviousNetPrice)) AND NOT(isnull(PreviousListAmount)) AND NOT(isnull(PreviousNetAmount))) {
                            if (lower(feeType) == "implementation"
                                OR lower(feeType) == "per occurrence"
                                OR lower(feeType) == "total") { //Added feetype = "total" condition by Khasim on 06/02/2026
                                adjustedListPrice = (listPrice - PreviousListPrice) * (aCVTCVPercentContriImpl / 100.0);
                                adjustedNetPrice = (netPrice - PreviousNetPrice) * (aCVTCVPercentContriImpl / 100.0);
                                // adjustedExtendedListPrice = (round(listAmount, 2) - PreviousListAmount) * (aCVTCVPercentContriImpl / 100.0);
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = (round(listAmount, listAmountRounding) - PreviousListAmount) * (aCVTCVPercentContriImpl / 100.0);
                                adjustedExtendedNetPrice = (round(netAmount, netAmountRounding) - PreviousNetAmount) * (aCVTCVPercentContriImpl / 100.0);
                            }
                            elif(lower(feeType) == "license") {
                                adjustedListPrice = (listPrice - PreviousListPrice) * (aCVTCVPercentContriLicense / 100.0);
                                adjustedNetPrice = (netPrice - PreviousNetPrice) * (aCVTCVPercentContriLicense / 100.0);
                                //adjustedExtendedListPrice = (round(listAmount, 2) - PreviousListAmount) * (aCVTCVPercentContriLicense / 100.0);
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = (round(listAmount, listAmountRounding) - PreviousListAmount) * (aCVTCVPercentContriLicense / 100.0);
                                adjustedExtendedNetPrice = (round(netAmount, netAmountRounding) - PreviousNetAmount) * (aCVTCVPercentContriLicense / 100.0);
                            }
                            elif(lower(feeType) == "monthly"
                                OR lower(feeType) == "years") { // Added by Chandu Pendyala for Years Fee type and Billing Frequency 9/23/2025
                                adjustedListPrice = (listPrice - PreviousListPrice) * (aCVTCVPercentContriMonthly / 100.0);
                                adjustedNetPrice = (netPrice - PreviousNetPrice) * (aCVTCVPercentContriMonthly / 100.0);
                                //adjustedExtendedListPrice = (round(listAmount, 2) - PreviousListAmount) * (aCVTCVPercentContriMonthly / 100.0);
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = (round(listAmount, listAmountRounding) - PreviousListAmount) * (aCVTCVPercentContriMonthly / 100.0);
                                adjustedExtendedNetPrice = (round(netAmount, netAmountRounding) - PreviousNetAmount) * (aCVTCVPercentContriMonthly / 100.0);
                            }
                            else {
                                adjustedListPrice = listPrice - PreviousListPrice;
                                adjustedNetPrice = netPrice - PreviousNetPrice;
                                //adjustedExtendedListPrice = round(listAmount, 2) - PreviousListAmount;
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = round(listAmount, listAmountRounding) - PreviousListAmount;
                                adjustedExtendedNetPrice = round(netAmount, netAmountRounding) - PreviousNetAmount;
                            }
                        } else {
                            if (lower(feeType) == "implementation"
                                OR lower(feeType) == "per occurrence"
                                OR lower(feeType) == "total") { //Added feetype = "total" condition by Khasim on 06/02/2026
                                adjustedListPrice = (listPrice + additionalACV) * (aCVTCVPercentContriImpl / 100.0);
                                adjustedNetPrice = (netPrice + additionalACV) * (aCVTCVPercentContriImpl / 100.0);
                                //adjustedExtendedListPrice = (round(listAmount, 2) + additionalACV) * (aCVTCVPercentContriImpl / 100.0);
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = (round(listAmount, listAmountRounding) + additionalACV) * (aCVTCVPercentContriImpl / 100.0);
                                adjustedExtendedNetPrice = (round(netAmount, netAmountRounding) + additionalACV) * (aCVTCVPercentContriImpl / 100.0);
                            }
                            elif(lower(feeType) == "license") {
                                adjustedListPrice = (listPrice + additionalACV) * (aCVTCVPercentContriLicense / 100.0);
                                adjustedNetPrice = (netPrice + additionalACV) * (aCVTCVPercentContriLicense / 100.0);
                                //adjustedExtendedListPrice = (round(listAmount, 2) + additionalACV) * (aCVTCVPercentContriLicense / 100.0);
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = (round(listAmount, listAmountRounding) + additionalACV) * (aCVTCVPercentContriLicense / 100.0);
                                adjustedExtendedNetPrice = (round(netAmount, netAmountRounding) + additionalACV) * (aCVTCVPercentContriLicense / 100.0);
                            }
                            elif(lower(feeType) == "monthly"
                                OR lower(feeType) == "years") { // Added by Chandu Pendyala for Years Fee type and Billing Frequency 9/23/2025
                                adjustedListPrice = (listPrice + additionalACV) * (aCVTCVPercentContriMonthly / 100.0);
                                adjustedNetPrice = (netPrice + additionalACV) * (aCVTCVPercentContriMonthly / 100.0);
                                // adjustedExtendedListPrice = (round(listAmount, 2) + additionalACV) * (aCVTCVPercentContriMonthly / 100.0);
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = (round(listAmount, listAmountRounding) + additionalACV) * (aCVTCVPercentContriMonthly / 100.0);
                                adjustedExtendedNetPrice = (round(netAmount, netAmountRounding) + additionalACV) * (aCVTCVPercentContriMonthly / 100.0);
                            }
                            else {
                                adjustedListPrice = listPrice + additionalACV;
                                adjustedNetPrice = netPrice + additionalACV;
                                //adjustedExtendedListPrice = round(listAmount, 2) + additionalACV;
                                // Added by Ashok Kumar: Feb 10
                                adjustedExtendedListPrice = round(listAmount, listAmountRounding) + additionalACV;
                                adjustedExtendedNetPrice = round(netAmount, netAmountRounding) + additionalACV;
                            }
                        }
                    }
                    /*Special TCV Calculation : Start*/
                    if (tCVType <> "") {
                        specialTCVCalcRequestJson = json();
                        jsonput(specialTCVCalcRequestJson, "tCVType", tCVType);
                        jsonput(specialTCVCalcRequestJson, "sKUSpecialTCVInfoJsonArray", sKUSpecialTCVInfoJsonArray);
                        jsonput(specialTCVCalcRequestJson, "tCVTierIDAttr", tCVTierIDAttr);
                        jsonput(specialTCVCalcRequestJson, "tCVQtyAttr", tCVQtyAttr);
                        jsonput(specialTCVCalcRequestJson, "customDiscountType", customDiscountType);
                        jsonput(specialTCVCalcRequestJson, "discountPercentage", discountPercentage);
                        jsonput(specialTCVCalcRequestJson, "listPrice", listPrice);
                        jsonput(specialTCVCalcRequestJson, "netPrice", netPrice);
                        jsonput(specialTCVCalcRequestJson, "listAmount", listAmount);
                        jsonput(specialTCVCalcRequestJson, "netAmount", netAmount);
                        specialTCVCalcResponseJson = util.util_specialTCVCalculation(specialTCVCalcRequestJson);
                        adjustedListPrice = jsonget(specialTCVCalcResponseJson, "adjustedListPrice", "float", adjustedListPrice);
                        adjustedNetPrice = jsonget(specialTCVCalcResponseJson, "adjustedNetPrice", "float", adjustedNetPrice);
                        adjustedExtendedListPrice = jsonget(specialTCVCalcResponseJson, "adjustedExtendedListPrice", "float", adjustedExtendedListPrice);
                        adjustedExtendedNetPrice = jsonget(specialTCVCalcResponseJson, "adjustedExtendedNetPrice", "float", adjustedExtendedNetPrice);
                    }
                    /*Special TCV Calculation : End*/
                    sbappend(retstr, each, "~adjustedListPrice_l_c~", string(adjustedListPrice), "|");
                    sbappend(retstr, each, "~adjustedNetPrice_l_c~", string(adjustedNetPrice), "|");
                    sbappend(retstr, each, "~adjustedExtendedListPrice_l_c~", string(adjustedExtendedListPrice), "|");
                    sbappend(retstr, each, "~adjustedExtendedNetPrice_l_c~", string(adjustedExtendedNetPrice), "|");
                    //Logic to calculate RollupPrices - Moved to util_rollupPricesCalculation library
                    if (rollupSKU <> ""
                        AND rollupSKU <> "NA") {
                        rollupCalcRequestJson = json();
                        jsonput(rollupCalcRequestJson, "rollupSKU", rollupSKU);
                        jsonput(rollupCalcRequestJson, "modelDocNum", modelDocNum);
                        jsonput(rollupCalcRequestJson, "skuNum", skuNum);
                        jsonput(rollupCalcRequestJson, "each", each);
                        jsonput(rollupCalcRequestJson, "MinimumCalcGrpSKUID", MinimumCalcGrpSKUID);
                        jsonput(rollupCalcRequestJson, "isUnitRate", isUnitRate);
                        jsonput(rollupCalcRequestJson, "typeOfTier", typeOfTier);
                        jsonput(rollupCalcRequestJson, "typeOfPricing", typeOfPricing);
                        jsonput(rollupCalcRequestJson, "changedQty", changedQty);
                        jsonput(rollupCalcRequestJson, "multiplyFactor", multiplyFactor);
                        jsonput(rollupCalcRequestJson, "previousQtyFactor", previousQtyFactor);
                        jsonput(rollupCalcRequestJson, "isAggregateHoldingSKU", isAggregateHoldingSKU);
                        jsonput(rollupCalcRequestJson, "feeType", feeType);
                        jsonput(rollupCalcRequestJson, "netAmount", netAmount);
                        jsonput(rollupCalcRequestJson, "netPrice", netPrice);
                        jsonput(rollupCalcRequestJson, "listAmount", listAmount);
                        jsonput(rollupCalcRequestJson, "setToNoUpdateAvailable", setToNoUpdateAvailable);
                        jsonput(rollupCalcRequestJson, "formulaSKUsetToNoUpdate", formulaSKUsetToNoUpdate);
                        jsonput(rollupCalcRequestJson, "txnType", txnType);
                        jsonput(rollupCalcRequestJson, "proposalTierDisplayDataForProposal", proposalTierDisplayDataForProposal);
                        jsonput(rollupCalcRequestJson, "propDisplayFormat", propDisplayFormat);
                        jsonput(rollupCalcRequestJson, "tierCalc", tierCalc);
                        jsonput(rollupCalcRequestJson, "uOM", uOM);
                        jsonput(rollupCalcRequestJson, "currentTierIndexForMonthlyFee", currentTierIndexForMonthlyFee);
                        jsonput(rollupCalcRequestJson, "bomLevel2DocNumber", bomLevel2DocNumber);
                        jsonput(rollupCalcRequestJson, "parentDocNumber", parentDocNumber);
                        jsonput(rollupCalcRequestJson, "lineDataJson", lineDataJson);
                        jsonput(rollupCalcRequestJson, "finalPriceJson", finalPriceJson);
                        jsonput(rollupCalcRequestJson, "rollUpSKUJson", rollUpSKUJson);
                        jsonput(rollupCalcRequestJson, "skuNumJson", skuNumJson);
                        rollupCalcResponseJson = util.util_rollupPricesCalculation(rollupCalcRequestJson);
                        finalPriceJson = jsonget(rollupCalcResponseJson, "finalPriceJson", "json", finalPriceJson);
                        rollUpSKUJson = jsonget(rollupCalcResponseJson, "rollUpSKUJson", "json", rollUpSKUJson);
                        rollupResponseKeys = jsonkeys(rollupCalcResponseJson);
                        for rollupKey in rollupResponseKeys {
                            if (startswith(rollupKey, "actionCodeUpdate_")) {
                                sbappend(retstr, substring(rollupKey, 17), "~actionCodeForCRM_l_c~", jsonget(rollupCalcResponseJson, rollupKey, "string", ""), "|");
                            }
                            elif(startswith(rollupKey, "customActionUpdate_")) {
                                sbappend(retstr, substring(rollupKey, 19), "~customActionCode_l_c~", jsonget(rollupCalcResponseJson, rollupKey, "string", ""), "|");
                            }
                            elif(startswith(rollupKey, "parentDictUpdate_")) {
                                put(parentDict, substring(rollupKey, 17), jsonget(rollupCalcResponseJson, rollupKey, "string", ""));
                            }
                        }
                    }
                    //end - RollupPrices

                    /*if (typeOfTier <> "Graduated Tier") {//Commented on 12May25 by Ashok
                        sbappend(retstr, each, "~feeCalculationMethod_l_c~Static|");
                        } else {
                        sbappend(retstr, each, "~feeCalculationMethod_l_c~Dynamic|");
                    }*/

                    sbappend(retstr, each, "~formulaInputSKUs_l_c~", formulaInputSKUs, "|");
                    sbappend(retstr, each, "~rollupSKU_l_c~", rollupSKU, "|");
                    sbappend(retstr, each, "~minimumCalcGrpSKUID_l_c~", MinimumCalcGrpSKUID, "|"); //Added: Bibin Sajeevan :16 Aug 2024:
                    sbappend(retstr, each, "~minimumFeeApplied_l_c~", string(minApplying), "|"); //Added: Bibin Sajeevan :16 Dec 2024:
                    //sbappend(retstr, each, "~listAmount_l~", string(round(listAmount, 2)), "|");

                    // Added:  Ashok Kumar : Feb 10
                    if (listAmount > -1 AND listAmount < 1) {
                        sbappend(retstr, each, "~displayExtendedListPrice_l_c~", (util.util_formatCurrency(listAmount, "USD", 7)), "|");
                        //sbappend(retstr, each, "~listAmount_l~", substring((util.util_formatCurrency(listAmount, "USD", 7)),1), "|");
                        sbappend(retstr, each, "~listAmount_l~", string(round(listAmount, 7)), "|");

                    } else {
                        sbappend(retstr, each, "~displayExtendedListPrice_l_c~", (util.util_formatCurrency(listAmount, "USD", 2)), "|");
                        sbappend(retstr, each, "~listAmount_l~", string(round(listAmount, 2)), "|");

                    }
                    //End : Ashok Kumar

                    if (extendedInvoiceMatchRate > 0) {
                        sbappend(retstr, each, "~extendedInvoiceMatchRate_l_c~", string(round(extendedInvoiceMatchRate, 2)), "|");
                    }


                    sbappend(retstr, each, "~netAmount_l~", string(round(netAmount, 2)), "|");
                    sbappend(retstr, each, "~customDiscountAmount_l~", string(listAmount - netAmount), "|");
                    //Included below logic to consider Cost and Margin for Price Calculation of Managed Solution Products - Chandu.P - 3/25/2026
                    if (cost > 0) {
                        sbappend(retstr, each, "~unitCost_l~", string(cost), "|");
                    }
                    if (margin > 0) {
                        sbappend(retstr, each, "~marginPercent_l~", string(round(margin, 2)), "|");
                    }
                    /*
                        if (feeType == "Implementation") {
                        sbappend(retstr, each, "~adjustedExtendedListPrice_l_c~", string((round(listAmount, 2) + additionalACV) * (aCVTCVPercentContriImpl / 100.0)), "|");
                        sbappend(retstr, each, "~adjustedExtendedNetPrice_l_c~", string((round(netAmount, 2) + additionalACV) * (aCVTCVPercentContriImpl / 100.0)), "|");
                        }
                        elif(feeType == "License") {
                        sbappend(retstr, each, "~adjustedExtendedListPrice_l_c~", string((round(listAmount, 2) + additionalACV) * (aCVTCVPercentContriLicense / 100.0)), "|");
                        sbappend(retstr, each, "~adjustedExtendedNetPrice_l_c~", string((round(netAmount, 2) + additionalACV) * (aCVTCVPercentContriLicense / 100.0)), "|");
                        }
                        elif(feeType == "Monthly") {
                        sbappend(retstr, each, "~adjustedExtendedListPrice_l_c~", string((round(listAmount, 2) + additionalACV) * (aCVTCVPercentContriMonthly / 100.0)), "|");
                        sbappend(retstr, each, "~adjustedExtendedNetPrice_l_c~", string((round(netAmount, 2) + additionalACV) * (aCVTCVPercentContriMonthly / 100.0)), "|");
                        }
                    */
                    if (tierCalc == "Term") {
                        sbappend(retstr, each, "~typeOfTierForCRM_l_c~Term|");
                    } else {
                        sbappend(retstr, each, "~typeOfTierForCRM_l_c~Quantity|");
                    }
                    //Uncommenting code on 23 Apr 2025-- to enable scenario 2 of license increase
                    //updated condition to check if the previous quantity is already set in Pricing Library, then it should not set in pricing engine.
                    if (changedQty > 0.0 AND NOT(PrevQtySetFlag)) {
                        sbappend(retstr, each, "~previousQtyFactor_l_c~", string(integer(changedQty)), "|");
                    }
                    elif(NOT(PrevQtySetFlag)) {
                        sbappend(retstr, each, "~previousQtyFactor_l_c~", string(integer(multiplyFactor)), "|");
                    }
                    //Uncommenting code on 23 Apr 2025-- to enable scenario 2 of license increase
                    // renewal build 4/17
                    //sbappend(retstr, each, "~previousQtyFactor_l_c~", string(integer(multiplyFactor)), "|");
                    if (displayMinPrice <> "") {
                        sbappend(retstr, each, "~listFeeMinimumDisplay_l_c~", displayMinPrice, "|");
                    } else {
                        sbappend(retstr, each, "~listFeeMinimumDisplay_l_c~", util.util_formatCurrency(minValueBeforeDiscount, "USD", 0), "|");
                        //sbappend(retstr, each, "~listFeeMinimumDisplay_l_c~", util.util_formatCurrency(newMinimumVal1, "USD", 0), "|");
                    }
                    sbappend(retstr, each, "~monthlyMinimum_l_c~", string(newMinimumVal1), "|");
                    sbappend(retstr, each, "~listFeeMinimum_l_c~", string(minValueBeforeDiscount), "|");

                    //sbappend(retstr, each, "~billingFrequency_l_c~", billingFrequency, "|");
                    sbappend(retstr, each, "~proposalDisplayFormat_l_c~", propDisplayFormat, "|");
                    //Navya: Added below code for billing integration - Start
                    /*
                    if (startswith(propDisplayFormat,"QRT")) {
                    sbappend(retstr, each, "~hasQRT_l_c~true|");
                    } else {
                    sbappend(retstr, each, "~hasQRT_l_c~false|");
                    }*/
                    //Billing integration code - End
                    //End : populating basic line level attriute's
                    tempFinalJson = jsonget(finalPriceJson, skuNum + "**" + modelDocNum, "json", json());
                    jsonput(tempFinalJson, "listPrice", listPrice);
                    jsonput(tempFinalJson, "listAmount", listAmount);
                    jsonput(tempFinalJson, "netPrice", netPrice);
                    jsonput(tempFinalJson, "netAmount", netAmount);
                    jsonput(tempFinalJson, "minApplying", minApplying);
                    jsonput(tempFinalJson, "minValue", string(minValue));
                    jsonput(tempFinalJson, "OriginalMinValue", string(minValueBeforeDiscount));
                    if (invoiceMatchRateMinimum > 0) {
                        jsonput(tempFinalJson, "OriginalMinValue", string(invoiceMatchRateMinimum));
                    }
                    jsonput(tempFinalJson, "DiscountPerOnMin", round(minValueDiscountPercent, 1));
                    jsonput(tempFinalJson, "nextLowerBound", nextLowerBound);
                    //jsonput(tempFinalJson, "previousQtyFactor", previousQtyFactor); Porposal Not generating properly making it as integer while addding to json
                    jsonput(tempFinalJson, "previousQtyFactor", integer(previousQtyFactor)); //Added : Bibin Sajeevan : 25 April 2025 : PreviousQtyFactor
                    if (typeOfTier == "Graduated Tier"
                        OR typeOfTier == "Two Factored Graduated Tier") {
                        jsonput(tempFinalJson, "TypeOfTier", typeOfTier); //Added : Bibin Sajeevan : 14 May 2025 : Type of Tier
                    }
                    //Two Factored Graduted Tier Pattern Changes AMTNB : ATM Content MAnagement
                    if (skuPrimaryAttrFlag) {
                        jsonput(tempFinalJson, "skuPrimaryAttrQty", integer(skuPrimaryAttrQty));
                    }
                    jsonput(tempFinalJson, "tierDisplayType", tierDisplayType);
                    jsonput(tempFinalJson, "TierDiscApplicable", tierDiscountApplicable);
                    jsonput(finalPriceJson, skuNum + "**" + modelDocNum, tempFinalJson);
                    jsonput(lineLevelTextAreaJson, "feeType", feeType);
                    jsonput(lineLevelTextAreaJson, "discountingNotAvailabel", discountingNotAvailabel);
                    jsonput(lineLevelTextAreaJson, "tieredMinimum", tieredMinimumJsonArray);
                    if (minTierUpdated == false) {
                        jsonput(lineLevelTextAreaJson, "originalTieredMinimum", tieredMinimumJsonArray);
                    }
                    jsonput(lineLevelTextAreaJson, "MinimumCalcType", minimumCalcType);

                    //Navya: Below code is added for a new pattern in Payment One - Start
                    if (minContributorSKU <> ""
                        AND MinContributionPer <> 0.0) {
                        tempjson = jsonget(minimumCalcJson, minContributorSKU + "**" + modelDocNum, "json", json());
                        jsonput(tempjson, "MinContributionPer", MinContributionPer);
                        jsonput(tempjson, "groupfixedMinimum", GroupFixedMinVal);
                        SumofListPrice = jsonget(tempjson, "SumofListPrice", "float", 0.0) + listPrice;
                        jsonput(tempjson, "SumofListPrice", SumofListPrice);
                        SumofNetPrice = jsonget(tempjson, "SumofNetPrice", "float", 0.0) + netPrice;
                        jsonput(tempjson, "SumofNetPrice", SumofNetPrice);
                        SumoflistAmount = jsonget(tempjson, "SumoflistAmount", "float", 0.0) + listAmount;
                        jsonput(tempjson, "SumoflistAmount", SumoflistAmount);
                        SumofNetAmount = jsonget(tempjson, "SumofNetAmount", "float", 0.0) + netAmount;
                        jsonput(tempjson, "SumofNetAmount", SumofNetAmount);
                        jsonput(tempjson, "feeType", jsonpathgetsingle(pricingJson, "$." + minContributorSKU + ".FeeType", "string", ""));
                        jsonput(minimumCalcJson, minContributorSKU + "**" + modelDocNum, tempjson);
                    }

                    //Projected TCV Calculation
                    if (billingFrequency == "Monthly") {
                        aPEPercentVal = 0.0; //Aditi:16July : If APE percent is not defined consider it as 0%
                        if (isnumber(aPEPercent)) {
                            aPEPercentVal = atof(aPEPercent);
                        }
                        if (isnumber(contractTerm)) {
                            contractTermVal = atoi(contractTerm);
                            netAmountVal = netAmount;
                            if (contractTermVal % 12 == 0) {
                                indices = range(contractTermVal / 12);
                                for indice in indices {
                                    projectedTCV = projectedTCV + (12 * netAmountVal);
                                    netAmountVal = netAmountVal + (netAmountVal * aPEPercentVal * .01);
                                }
                            }
                            elif(contractTermVal % 12 <> 0 AND contractTermVal > 12) {
                                indices = range((contractTermVal / 12) + 1);
                                lastIndex = indices[sizeofarray(indices) - 1];
                                for indice in indices {
                                    if (indice <> lastIndex) {
                                        projectedTCV = projectedTCV + (12 * netAmountVal);
                                        netAmountVal = netAmountVal + (netAmountVal * aPEPercentVal * .01);
                                    } else {
                                        projectedTCV = projectedTCV + ((contractTermVal % 12) * netAmountVal);
                                    }
                                }
                            }
                            elif(contractTermVal < 12) {
                                projectedTCV = projectedTCV + (netAmountVal * contractTermVal);
                            }
                        }
                    }
                    elif(billingFrequency == "Years"
                        AND feeType == "Years") { //Pranavya 22-Aug-2025: Adding the Block of code to consider the new pattern with billing frequency as 'Years'.
                        aPEPercentVal = 0.0;
                        if (isnumber(aPEPercent)) {
                            aPEPercentVal = atof(aPEPercent);
                        }
                        if (isnumber(contractTerm)) {
                            contractTermVal = atoi(contractTerm);
                            netAmountVal = netAmount;
                            indices = range(contractTermVal);
                            for indice in indices {
                                projectedTCV = projectedTCV + (netAmountVal);
                                netAmountVal = netAmountVal + (netAmountVal * aPEPercentVal * .01);
                            }
                        }
                    }
                    elif(billingFrequency == "Years") { //Pranavya 22-Aug-2025: Adding the Block of code to consider the new pattern with billing frequency as 'Years'.
                        aPEPercentVal = 0.0;
                        if (isnumber(aPEPercent)) {
                            aPEPercentVal = atof(aPEPercent);
                        }
                        if (isnumber(contractTerm)) {
                            contractTermVal = ceil(atoi(contractTerm) / 12);
                            netAmountVal = netAmount;
                            indices = range(integer(contractTermVal));
                            for indice in indices {
                                projectedTCV = projectedTCV + (netAmountVal);
                                netAmountVal = netAmountVal + (netAmountVal * aPEPercentVal * .01);
                            }
                        }
                    }
                    elif(billingFrequency == "One Time") {
                        projectedTCV = projectedTCV + netAmount;
                        sbappend(retstr, each, "~contractTerm_l_c~1|");
                        endDate = datetostr(addmonths(strtojavadate(contractStartDate, "MM/dd/yyyy HH:mm:ss"), 1));
                        sbappend(retstr, each, "~contractEndDate_l~", endDate, "|");
                    }
                }
                jsonput(lineLevelTextAreaJson, "TierDiscApplicable", tierDiscountApplicable);
                jsonput(lineLevelTextAreaJson, "DiscountApplicable", discountApplicable);
                if (isMultiTierDisplay == true) { //Aditi
                    sbappend(retstr, each, "~tierDisplay_l_c~", "Yes", "|");
                }
                if (isUnitRate AND typeOfTier <> "Graduated Tier"
                    AND typeOfTier <> "Two Factored Graduated Tier") {
                    //if (changedQty > 0.0) {
                    if (typeOfPricing == "Formula"
                        AND isUnitRate == true AND typeOfTier <> "Fixed Fee") {
                        sbappend(retstr, each, "~_price_quantity~", string(integer(changedQty)), "|");
                    } else {
                        sbappend(retstr, each, "~_price_quantity~", string(integer(multiplyFactor)), "|");
                    }
                }
                elif(multiplyFactor == 0.0) {
                    sbappend(retstr, each, "~_price_quantity~", string(integer(multiplyFactor)), "|");
                }
                //Navya: Adding below code for billing integration - Start
                /*if(typeOfTier == "Static Tier" OR typeOfTier == "Graduated Tier"){
                    sbappend(retstr, each, "~typeOfTier_l_c~", typeOfTier, "|");
                }*/
                if (billingFrequency == "Monthly") {
                    associatedOneTimeSKUs = jsonget(oneTimeSKUAssociationJson, rollupSKU, "string", "");
                    sbappend(retstr, each, "~oneTimeAssociation_l_c~", associatedOneTimeSKUs, "|");
                }
                /* Commenting Code 16 June 2025, Below written new code
                    if (minimumCalcType == "individual"
                    OR minimumCalcType == "group") {
                    if (minimumCalcType == "individual") {
                    sbappend(retstr, each, "~minimumType_l_c~Individual|");
                    } else {
                    sbappend(retstr, each, "~minimumType_l_c~Group|");
                    }
                    
                    }
                */
                if (minimumCalcType == "individual") {
                    sbappend(retstr, each, "~minimumType_l_c~Individual|");
                }
                elif(minimumCalcType == "tiered individual") {
                    //sbappend(retstr, each, "~minimumType_l_c~Tiered Individual|");
                    sbappend(retstr, each, "~minimumType_l_c~Individual|");
                }
                elif(minimumCalcType == "tieredramp individual") {
                    //sbappend(retstr, each, "~minimumType_l_c~TieredRamp Individual|");
                    sbappend(retstr, each, "~minimumType_l_c~Individual|");
                }
                elif(minimumCalcType == "tieredramprunrate") {
                    //sbappend(retstr, each, "~minimumType_l_c~TieredRampRunrate|");
                    sbappend(retstr, each, "~minimumType_l_c~Individual|");
                }
                elif(minimumCalcType == "group") {
                    sbappend(retstr, each, "~minimumType_l_c~Group|");
                }
                //Navya: Billing integration code change - End
                sbappend(retstr, each, "~customDiscountType_l~", customDiscountType, "|");
                sbappend(retstr, each, "~customDiscountValue_l~", customDiscountValue, "|");
                sbappend(retstr, each, "~minimumDiscountType_l_c~", minimumDiscountType, "|");
                sbappend(retstr, each, "~minimumDiscount_l_c~", string(minimumDiscountValue), "|");
                sbappend(retstr, each, "~isGroupMinimumContributor_l_c~", string(isGroupMinimumContributor), "|");
                sbappend(retstr, each, "~isAggregatePricingSKU_l_c~", string(isAggregatePricingSKU), "|");
                sbappend(retstr, each, "~isTierPricingSKU_l_c~", string(isTierPricingSKU), "|");
                if (findinarray(cQFSKUsArr, skuNum) == -1) { //UOM for CQF is getting set by user selection on UI.
                    sbappend(retstr, each, "~requestedUnitOfMeasure_l~", uOM, "|");
                }
                sbappend(retstr, each, "~feeType_l_c~", feeType, "|");
                sbappend(retstr, each, "~billingFrequency_l_c~", billingFrequency, "|");
                sbappend(retstr, each, "~cRMACVTCVExclusionFlag_l_c~", string(crmACVTCVExclusionFlag), "|");
                //Chandu.P - 12/22/2025 - Updating the condition to skip for implementation SKU to show proper tier calculation for Implementation SKU for ATMNB Model.
                //if (typeOfPricing == "Formula"
                //    AND isUnitRate == true AND typeOfTier <> "Fixed Fee") {
                if (typeOfPricing == "Formula"
                    AND isUnitRate == true AND typeOfTier <> "Fixed Fee"
                    AND feeType <> "Implementation") {
                    sbappend(retstr, each, "~qtyFactor_l_c~", string(integer(changedQty)), "|");
                } else {
                    sbappend(retstr, each, "~qtyFactor_l_c~", string(integer(multiplyFactor)), "|");
                }
                //added by Shivam for license increase
                if (jsonget(skuNumJson, "setAboUpdate", "boolean", false)) {
                    //sbappend(retstr, each, "~oRCL_ABO_ActionCode_l~UPDATE|");
                    sbappend(retstr, each, "~customActionCode_l_c~UPDATE|");
                    if (findinarray(cQFSKUsArr, skuNum) <> -1) { // Marking CQF SKU as UPDATE In case if license increase
                        sbappend(retstr, each, "~oRCL_ABO_ActionCode_l~UPDATE|");
                    }
                }
                if (calculationExclusionFlag == "y"
                    OR crmACVTCVExclusionFlag == true) {
                    sbappend(retstr, each, "~calculationExclusionFlag_l_c~", calculationExclusionFlag, "|");
                } else {
                    //Added else condition because, when we create the quote and after that if we update the data table ACV TCV flag, 
                    //It will not reflect the latest dat table changes. 
                    sbappend(retstr, each, "~calculationExclusionFlag_l_c~|");
                }
                //Added : Bibin Sajeevan : 09 May 2025 : Setting Contract Term as 1 for One Time SKUs
                /* Shifting this up : Chirag 11 June 2025
                if(billingFrequency == "One Time") {
                sbappend(retstr, each, "~contractTerm_l_c~1|");
                endDate = datetostr(addmonths(strtojavadate(contractStartDate, "MM/dd/yyyy HH:mm:ss"), 1));
                sbappend(retstr, each, "~contractEndDate_l~", endDate, "|");
                }
                */
                //Added : Bibin Sajeevan : 09 May 2025 : Setting Contract Term as 1 for One Time SKUs
            }
            elif(findinarray(jsonkeys(groupMinimunJson), skuNum) == -1 AND findinarray(jsonkeys(aggregatePricingJson), skuNum) == -1 AND modelVarName <> "professionalServices_m") {
                sbappend(retstr, each, "~calculationExclusionFlag_l_c~y", "|");
                sbappend(retstr, each, "~cRMACVTCVExclusionFlag_l_c~true|");
            }
            lineDataJsonUpdate = jsonpathgetsingle(lineDataJson, "$." + each, "json", json());
            jsonput(lineDataJsonUpdate, "lineData", jsontostr(lineLevelTextAreaJson));
            jsonput(lineDataJson, each, lineDataJsonUpdate);
            sbappend(retstr, each, "~lineDataJson_l_c~", jsontostr(lineLevelTextAreaJson), "|");
            sbappend(retstr, each, "~aBOJson_l_c~", jsontostr(aboJson), "|");
        }

        //populating Projected TCV
        sbappend(retstr, "1~projectedTCV_t_c~", string(projectedTCV), "|");

        //setting value of "Needs Product Specialist Input" if any SKU is having $0 in Userinput price column
        sbappend(retstr, "1~needsProductSpecialistInput_t_c~", string(needsProductSpecialistInput), "|");
        sbappend(retstr, "1~miscFeeQuoteInput_t_c~", string(miscQuote), "|");

        //Seting Group Min on Option Class : Start
        groupMinReqJson = json();
        jsonput(groupMinReqJson, "groupMinimumJson", groupMinimunJson);
        jsonput(groupMinReqJson, "finalPriceJson", finalPriceJson);
        jsonput(groupMinReqJson, "lineDataJson", lineDataJson);
        jsonput(groupMinReqJson, "assumptionDataStr", assumptionDataStr);
        jsonput(groupMinReqJson, "signingBonusSkuJson", signingBonusSkuJson);
        jsonput(groupMinReqJson, "multiGroupMinJson", multiGroupMinJson); //AMTNC -- Norcross model pattern -- Subtotal under subtotal or group minimum under group minimum pattern
        groupMinResJson = util.util_groupMinimumFinalDataGeneration(groupMinReqJson);
        sbappend(retstr, jsonget(groupMinResJson, "returnString", "string", ""));
        finalPriceJson = jsonget(groupMinResJson, "finalPriceJson", "json", finalPriceJson);
        signingBonusSkuJson = jsonget(groupMinResJson, "signingBonusSkuJson", "json", json());
        lineDataJson = jsonget(groupMinResJson, "lineDataJson", "json", lineDataJson);
        //Seting Group Min on Option Class : End

        //Seting Aggregated Tier Display Arrayset on Option Class : Start
        aggregateReqJson = json();
        jsonput(aggregateReqJson, "aggregatePricingJson", aggregatePricingJson);
        jsonput(aggregateReqJson, "finalPriceJson", finalPriceJson);
        jsonput(aggregateReqJson, "lineDataJson", lineDataJson);
        aggregateUtilResponse = util.util_aggregateFinalDataGeneration(aggregateReqJson);
        sbappend(retstr, jsonget(aggregateUtilResponse, "returnString", "string", ""));
        finalPriceJson = jsonget(aggregateUtilResponse, "finalPriceJson", "json", finalPriceJson);
        lineDataJson = jsonget(aggregateUtilResponse, "lineDataJson", "json", lineDataJson);

        //Calling New util for PaymentOne New Pattern
        minContributionReqJson = json();
        jsonput(minContributionReqJson, "minimumCalcJson", minimumCalcJson);
        jsonput(minContributionReqJson, "finalPriceJson", finalPriceJson);
        jsonput(minContributionReqJson, "lineDataJson", lineDataJson);
        jsonput(minContributionReqJson, "signingBonusSkuJson", signingBonusSkuJson);
        minContributionResponse = util.util_getMinimumPrice(minContributionReqJson);
        sbappend(retstr, jsonget(minContributionResponse, "retStr", "string", ""));
        finalPriceJson = jsonget(minContributionResponse, "finalPriceJson", "json", finalPriceJson);
        //Seting Aggregated Tier Display Arrayset on Option Class : End
        rollUpJsonkeys = jsonkeys(rollUpSKUJson);
        totalMonthly = 0.0;
        totalOnetime = 0.0;
        sbappend(retStr, assumptionDataStr);
    }
    jsonput(returnJson, "updatedLineDataJson", lineDataJson);
    jsonput(returnJson, "attributeJson", attributeJson);
    parentRollUpArr = keys(parentDict);
    for val in parentRollUpArr {
        sbappend(retstr, parentDocNumber, "~totalRollupPrices_l_c~", get(parentDict, val), "|");
    }

    jsonput(returnJson, "finalPriceJson", finalPriceJson);
}
jsonput(returnJson, "result", sbtostring(retstr));
return returnJson;
