# 核心代码
## SME同步
```code
package com.ruijie.contract.service.impl;

import com.alibaba.fastjson.JSON;
import com.ruijie.contract.api.ProductDataMartApi;
import com.ruijie.contract.api.SMEApi;
import com.ruijie.contract.domain.ContractAttachment;
import com.ruijie.contract.domain.ContractDetail;
import com.ruijie.contract.domain.ContractInfo;
import com.ruijie.contract.domain.ContractProductInfo;
import com.ruijie.contract.domain.GetProductsRequestDTO;
import com.ruijie.contract.domain.GetProductsResponseDTO;
import com.ruijie.contract.domain.vo.HRERPStepDeptVO;
import com.ruijie.contract.domain.vo.HrUserVO;
import com.ruijie.contract.domain.vo.UserByRoleDeptVO;
import com.ruijie.contract.mapper.expand.ContractAttachmentExpandMapper;
import com.ruijie.contract.mapper.expand.ContractDetailExpandMapper;
import com.ruijie.contract.mapper.expand.ContractInfoExpandMapper;
import com.ruijie.contract.mapper.expand.ContractProductInfoExpandMapper;
import com.ruijie.contract.service.CommonInfoService;
import com.ruijie.contract.service.ContractInfoService;
import com.ruijie.contract.service.HrUserInfoService;
import com.ruijie.contract.service.SMEService;
import com.ruijie.contract.service.dto.FileUrlDTO;
import com.ruijie.contract.service.dto.SMESyncProcessRequestDTO;
import com.ruijie.contract.service.dto.SMESyncResponseDTO;
import com.ruijie.contract.service.dto.SubmitOrderToECPDTO;
import com.ruijie.contract.service.dto.ToCesOverSeasDetails;
import com.ruijie.contract.service.dto.ToCesOverSeasProduct;
import com.ruijie.contract.service.dto.ToEcpOverSeasAction;
import com.ruijie.contract.service.dto.ToEcpOverSeasEnclosure;
import com.ruijie.contract.service.dto.ToEcpOverSeasFormBase;
import com.ruijie.contract.service.dto.ToEcpOverSeasFormMain;
import com.ruijie.contract.service.dto.ToEcpOverSeasOpp;
import com.ruijie.contract.service.dto.ToEcpOverSeasTrade;
import com.ruijie.contract.util.CommonException;
import com.ruijie.contract.util.ContractProcessEnum;
import com.ruijie.contract.util.ContractTypeEnum;
import com.ruijie.framework.common.RemoteResult;

import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

import static com.ruijie.contract.controller.SMEController.ecpAesKey;

/**
 * @author zrh
 * @date 2020/7/28
 */
@Service
public class SMEServiceImpl implements SMEService {
    @Autowired
    ContractInfoExpandMapper contractInfoExpandMapper;
    @Autowired
    ContractProductInfoExpandMapper contractProductInfoExpandMapper;
    @Autowired
    ContractDetailExpandMapper contractDetailExpandMapper;
    @Autowired
    ContractAttachmentExpandMapper contractAttachmentExpandMapper;
    @Autowired
    HrUserInfoService hrUserInfoService;
    @Autowired
    ContractInfoService contractInfoService;
    @Autowired
    CommonInfoService commonInfoService;
    @Autowired
    ProductDataMartApi productDataMartApi;
    @Autowired
    SMEApi smeApi;
    private final String splitType = "福州";
    private final String customerContractType = "VAD";
    private final String accountPeriod = "0,100";
    private final String marketSegmentCode = "10115";
    private final String marketSegmentName = "SME（fixed price product）";
    private final String applicationScenariosCode = "10569";
    private final String applicationScenariosName = "VAD store";

    @Override
    @Transactional(rollbackFor = Exception.class)
    public SMESyncResponseDTO syncSMEContractInfo(SubmitOrderToECPDTO param) {
        SMESyncResponseDTO responseDTO = new SMESyncResponseDTO();
        ToEcpOverSeasFormBase overSeasFormBase = param.getOverSeasFormBase();
        ToEcpOverSeasFormMain overSeasFormMain = param.getOverSeasFormMain();
        ToCesOverSeasDetails overSeasDetails = param.getOverSeasDetails();
        ToEcpOverSeasOpp overSeasOpp = param.getOverSeasOpp();
        ToEcpOverSeasTrade overSeasTrade = param.getOverSeasTrade();
        List<ToCesOverSeasProduct> overSeasProduct = param.getOverSeasProduct();
        ToEcpOverSeasAction overSeasAction = param.getOverSeasAction();
        ContractDetail contractDetail = new ContractDetail();
        ContractInfo contractInfo = new ContractInfo();
        //合同基本信息
        saveContractInfo(overSeasFormBase, overSeasDetails, contractInfo);
        //合同明细
        saveContractDetail(overSeasFormBase, overSeasFormMain, overSeasDetails, overSeasTrade, contractDetail, contractInfo);
        //合同产品
        saveContractProduct(overSeasProduct, contractInfo);
        //组装返回值
        responseDTO.setFormID(contractInfo.getSmeFromId());
        responseDTO.setContractNum(contractInfo.getContractNum());
        responseDTO.setCurrentNode(ContractProcessEnum.PROCESS_CHANNEL_TODO.getMsg());
        responseDTO.setCurrentNodeCode(ContractProcessEnum.PROCESS_CHANNEL_TODO.getType());
        responseDTO.setNextNode(ContractProcessEnum.PROCESS_CHANNEL_CONFIRM.getMsg());
        responseDTO.setNextNodeCode(ContractProcessEnum.PROCESS_CHANNEL_CONFIRM.getType());
        return responseDTO;
    }

    @Override
    public void syncProcessNodeToSME(SMESyncProcessRequestDTO requestDTO){
        requestDTO.setOpinionTime(new Date());
        String requestStr = null;
        try {
            requestStr = RemoteResult.setData(JSON.toJSONString(requestDTO), false, false, ecpAesKey);
        } catch (Exception e) {
            throw new RuntimeException("加密失败");
        }
        smeApi.syncOrderStatusFromCes(RemoteResult.successResult(requestStr));
    }

    @Override
    public FileUrlDTO getFileUrlForCes(FileUrlDTO request) throws Exception {
        String requestStr = null;
        requestStr = RemoteResult.setData(JSON.toJSONString(request), false, false, ecpAesKey);
        RemoteResult<String> response = smeApi.getFileUrlForCes(RemoteResult.successResult(requestStr));
        String responseStr = RemoteResult.getData(response.getData(), false, false, ecpAesKey);
        FileUrlDTO fileUrlDTO = JSON.parseObject(responseStr,FileUrlDTO.class);
        return fileUrlDTO;
    }

    private void saveContractAttachment(ToEcpOverSeasFormBase overSeasFormBase, List<ToEcpOverSeasEnclosure> overSeasEnclosure, ContractInfo contractInfo) {
        overSeasEnclosure.stream().forEach(toEcpOverSeasEnclosure -> {
            ContractAttachment contractAttachment = new ContractAttachment();
            contractAttachment.setContractId(contractInfo.getId());
            contractAttachment.setFileName(toEcpOverSeasEnclosure.getDocumentEnclosureName());
            contractAttachment.setFileToken(toEcpOverSeasEnclosure.getFileServerId());
            contractAttachment.setIsDelete(false);
            contractAttachment.setCreateTime(new Date());
            contractAttachment.setCreateBy(overSeasFormBase.getCreatUserID());
            contractAttachmentExpandMapper.insertSelective(contractAttachment);
        });
    }

    private void saveContractProduct(List<ToCesOverSeasProduct> overSeasProduct, ContractInfo contractInfo) {
        List<String> productCodes = overSeasProduct.stream().map(ToCesOverSeasProduct::getProductCode).collect(Collectors.toList());
        GetProductsRequestDTO getProductsDTO = new GetProductsRequestDTO();
        getProductsDTO.setPageNum(1);
        getProductsDTO.setPageSize(9999);
        getProductsDTO.setProductCodes(productCodes.toArray(new String[productCodes.size()]));
        RemoteResult getProductsResult = productDataMartApi.getProducts(getProductsDTO);
        if (getProductsResult.isSuccess()) {
            Map<String,ContractProductInfo> productsResponseDTOMap = new HashMap<>();
            GetProductsResponseDTO result = JSON.parseObject(JSON.toJSONString(getProductsResult.getData()), GetProductsResponseDTO.class);
            List<GetProductsResponseDTO.ProductsResponseDTO> productsResponseDTOS = result.getList();
            productsResponseDTOS.stream().forEach((product) -> {
                ContractProductInfo productInfo = new ContractProductInfo();
                productInfo.setDbOrIncrement(StringUtils.isNotBlank(product.getDborincrementFlag())? Byte.valueOf(product.getDborincrementFlag()) : null);
                productInfo.setSoftFlag(product.getMaterialFlag());
                productInfo.setProductClassShort(product.getLineAbbreviate());
                productInfo.setProductCategoryCode(product.getSeriesId());
                productInfo.setProductCode(product.getProductCode());
                productInfo.setProductRetailPrice(product.getOverseasuserprice());
                productInfo.setProductRealCosts(product.getProductrealcosts());
                productInfo.setProductClassId(product.getLineId());
                productInfo.setProductType(product.getProducttype());
                productInfo.setProductAuthType(product.getProductpsctype());
                productInfo.setUnit(product.getProductUnit());
                //写入字典
                String key = productInfo.getProductCode()+productInfo.getProductClassShort()+productInfo.getProductCategoryCode();
                productsResponseDTOMap.put(key,productInfo);
            });
            AtomicInteger itemIndex = new AtomicInteger(1);
            overSeasProduct.stream().forEach((toEcpOverSeasProduct) -> {
                ContractProductInfo productInfo = new ContractProductInfo();
                productInfo.setContractId(contractInfo.getId());
                productInfo.setItemIndex(itemIndex.getAndIncrement());
                productInfo.setProductId(toEcpOverSeasProduct.getProductID() == null ? 0 : Integer.valueOf(toEcpOverSeasProduct.getProductID()));
                productInfo.setProductCode(toEcpOverSeasProduct.getProductCode());
                productInfo.setProductName(toEcpOverSeasProduct.getProductName());
                productInfo.setProductCategoryCode(toEcpOverSeasProduct.getProductCategoryID());
                productInfo.setProductCategoryName(toEcpOverSeasProduct.getProductCategoryName());
                productInfo.setCreateTime(new Date());
                //产品单价，拆分前金额，拆分后金额,显示金额，计算金额，使用分销价格
                productInfo.setBeforeSplitProductPriceSigle(toEcpOverSeasProduct.getDistributorPrice());
                productInfo.setDistributorPrice(toEcpOverSeasProduct.getDistributorPrice());
                productInfo.setProductPriceSigle(toEcpOverSeasProduct.getDistributorPrice());
                productInfo.setSplitPrice(toEcpOverSeasProduct.getDistributorPrice());
                productInfo.setBeSplitPrice(toEcpOverSeasProduct.getDistributorPrice());
                productInfo.setCalcPrice(toEcpOverSeasProduct.getDistributorPrice());
                productInfo.setDisplayPrice(toEcpOverSeasProduct.getDistributorPrice());
                //订单金额使用单价*数量
                productInfo.setOrderAmount(toEcpOverSeasProduct.getDistributorPrice().multiply(new BigDecimal(toEcpOverSeasProduct.getProductAmount())));
                //todo 实际结算金额
                productInfo.setSettlementAmount(null);
                productInfo.setProductNum(toEcpOverSeasProduct.getProductAmount());
                productInfo.setProductClassShort(toEcpOverSeasProduct.getProductClassShort());
                productInfo.setExpectedDeliveryDate(toEcpOverSeasProduct.getExpectedDeliveryDate());
                //填充产品详情，从字典表取出数据
                String key = productInfo.getProductCode()+productInfo.getProductClassShort()+productInfo.getProductCategoryCode();
                ContractProductInfo productInfoDetail = productsResponseDTOMap.get(key);
                if(productInfoDetail!=null) {
                    productInfo.setDbOrIncrement(productInfoDetail.getDbOrIncrement());
                    productInfo.setSoftFlag(productInfoDetail.getSoftFlag());
                    productInfo.setProductRetailPrice(productInfoDetail.getProductRetailPrice());
                    productInfo.setProductRealCosts(productInfoDetail.getProductRealCosts());
                    productInfo.setProductType(productInfoDetail.getProductType());
                    productInfo.setProductAuthType(productInfoDetail.getProductAuthType());
                    productInfo.setUnit(productInfoDetail.getUnit());
                    productInfo.setProductClassId(productInfoDetail.getProductClassId());
                    contractProductInfoExpandMapper.insertSelective(productInfo);
                }else {
                    throw new CommonException("未匹配到正确的产品");
                }
            });
        } else {
            throw new CommonException("调用产品详情失败");
        }
    }

    private void saveContractDetail(ToEcpOverSeasFormBase overSeasFormBase, ToEcpOverSeasFormMain overSeasFormMain, ToCesOverSeasDetails overSeasDetails, ToEcpOverSeasTrade overSeasTrade, ContractDetail contractDetail, ContractInfo contractInfo) {
        contractDetail.setContractId(contractInfo.getId());
        //供货单位固定值2 锐捷网络股份有限公司
        contractDetail.setSupplyUnitId(2);
        //订货单位使用渠道ID
        contractDetail.setOrderUnitCode(overSeasFormBase.getCreatUserID());
        //订货单位和最终客户取渠道名称
        contractDetail.setOrderUnit(overSeasFormBase.getCreatUserName());
        contractDetail.setFinalCustomer(overSeasFormBase.getCreatUserName());
        //签约类型VAD
        contractDetail.setCustomerContractType(customerContractType);
        //净销售额取订单金额
        contractDetail.setNetSales(overSeasDetails.getDistributorPriceTotal());
        //默认计量记点
        contractDetail.setMeteringPoint((byte) 1);
        //收款银行ID默认招商银行福州分行营业部
        contractDetail.setPaymentBankId(1);
        //账期默认0,100
        contractDetail.setAccountPeriod(accountPeriod);
        //订货单位在oracle中的编码暂时为空
        contractDetail.setBuyerCodeForOracle("");
        //外销区域
        UserByRoleDeptVO userByRoleDeptVO = commonInfoService.getUserByRoleDeptInfo(overSeasFormMain.getSalesmanID());
        contractDetail.setRegionCode(userByRoleDeptVO.getSaleAreaId());
        contractDetail.setRegionName(userByRoleDeptVO.getSaleAreaName());
        contractDetail.setDepartmentCode(userByRoleDeptVO.getDeptId());
        contractDetail.setDepartmentName(userByRoleDeptVO.getDeptName());
        //HR部门信息
        HrUserVO hrUserVO = hrUserInfoService.getUserInfo(userByRoleDeptVO.getUserCode());
        if (hrUserVO != null) {
            contractDetail.setHrDeptId(hrUserVO.getDeptCode());
            contractDetail.setHrDeptName(hrUserVO.getDeptName());
        }
        contractDetail.setOrderUnitTaxRate(null);
        contractDetail.setSalesManCode(overSeasDetails.getCashCreditType().toString());
        contractDetail.setSalesManName(convertSalesManName(overSeasDetails.getCashCreditType()));
        //细分市场固定值
        contractDetail.setMarketSegmentCode(marketSegmentCode);
        contractDetail.setMarketSegmentName(marketSegmentName);
        contractDetail.setApplicationScenariosCode(applicationScenariosCode);
        contractDetail.setApplicationScenariosName(applicationScenariosName);
        //外销所在 erp 二级部门
        HRERPStepDeptVO hrerpStepDeptVO = hrUserInfoService.getUserDeptInfo(userByRoleDeptVO.getUserCode());
        contractDetail.setSecondaryDepCode(hrerpStepDeptVO.getErpDeptId());
        contractDetail.setSecondaryDepName(hrerpStepDeptVO.getErpDeptName());
        //实际结算金额取订单金额
        contractDetail.setActualSettlementAmount(overSeasDetails.getDistributorPriceTotal());
        contractDetail.setSalesManCode(userByRoleDeptVO.getUserCode());
        contractDetail.setSalesManName(userByRoleDeptVO.getUserName());
        contractDetail.setPreSalesCode(overSeasFormMain.getPreSaleID());
        contractDetail.setPreSalesName(overSeasFormMain.getPreSaleName());
        contractDetail.setSoldCountry(overSeasFormMain.getOverSeasSalesCountry());
        contractDetail.setPowerCordSpecification(overSeasFormMain.getPowerCord());
        contractDetail.setTradeTerm(overSeasFormMain.getTradeTerms());
        contractDetail.setCurrencyId(overSeasFormMain.getFormCurrencyId());
        contractDetail.setCurrencyName(overSeasFormMain.getFormCurrency());
        contractDetail.setIsRebate(overSeasDetails.getIsUseRebate());
        contractDetail.setRebateAmount(overSeasDetails.getRebateAmount());
        contractDetail.setSalesType(overSeasDetails.getCashCreditType() == null ? null : (byte) overSeasDetails.getCashCreditType().intValue());
        contractDetail.setRebate(overSeasDetails.getIsUseRebate() ? new BigDecimal(0.98) : new BigDecimal(1));
        contractDetail.setOrderSales(overSeasDetails.getDistributorPriceTotal());
        contractDetail.setOverseasRate(overSeasDetails.getOverseasRate());
        contractDetail.setOverseasRateToUsd(new BigDecimal(1).divide(overSeasDetails.getOverseasRate()));
        contractDetail.setModeOfShipping(overSeasTrade.getTransportMode());
        contractDetailExpandMapper.insertSelective(contractDetail);
    }

    private void saveContractInfo(ToEcpOverSeasFormBase overSeasFormBase, ToCesOverSeasDetails overSeasDetails, ContractInfo contractInfo) {
        contractInfo.setContractStatus((byte) 0);
        contractInfo.setContractProcessStatus((byte) 2);
        contractInfo.setEcpPoDocumentId(UUID.randomUUID().toString());
        String contractNum = contractInfoService.createContractNum(splitType, ContractTypeEnum.CONTRACT_TYPE_INTER.getType());
        contractInfo.setContractNum(contractNum);
        //合同类型固定国际业务下单
        contractInfo.setContractType(ContractTypeEnum.CONTRACT_TYPE_INTER.getType());
        //客戶合同编号->合同编号
        contractInfo.setCustomerContractNum(contractNum);
        contractInfo.setSmeFromId(overSeasFormBase.getFormID());
        contractInfo.setProjectName(overSeasFormBase.getDocumentTitle());
        contractInfo.setAgentId(overSeasFormBase.getCreatUserID());
        contractInfo.setCreateById(overSeasFormBase.getCreatUserID());
        contractInfo.setCreateByName(overSeasFormBase.getCreatUserName());
        //合同金额使用订单金额
        contractInfo.setContractSum(overSeasDetails.getDistributorPriceTotal());
        contractInfo.setCreateTime(new Date());
        contractInfo.setUpdateTime(new Date());
        contractInfo.setContractTypeCode(contractInfoService.getContractTypeCode(ContractTypeEnum.CONTRACT_TYPE_INTER.getType()));
        contractInfoExpandMapper.insertSelectiveKey(contractInfo);
    }

    private String convertSalesManName(Integer code) {
        if (code == null) {
            return null;
        }
        return code.equals(1) ? "现销" : "赊销";
    }
}

```
## 电汇单
```
package com.ruijie.contract.service.impl;

import com.github.pagehelper.PageHelper;
import com.github.pagehelper.PageInfo;
import com.ruijie.contract.domain.ContractAssign;
import com.ruijie.contract.domain.ContractInfo;
import com.ruijie.contract.domain.IntelContractAssign;
import com.ruijie.contract.domain.TelegraphicTransferInternation;
import com.ruijie.contract.domain.WaterOrderAttachment;
import com.ruijie.contract.domain.vo.AddAssignTelegraphicTransferInternationVO;
import com.ruijie.contract.domain.vo.AddTelegraphicTransferInternationVO;
import com.ruijie.contract.domain.vo.AssignTelegraphicTransferInternationVO;
import com.ruijie.contract.domain.vo.ContractAssignVO;
import com.ruijie.contract.domain.vo.ListTelegraphicTransferInternationVO;
import com.ruijie.contract.domain.vo.TelegraphicConfirmCollectionVO;
import com.ruijie.contract.domain.vo.TelegraphicTransferInternationDetailVO;
import com.ruijie.contract.domain.vo.TelegraphicTransferInternationVO;
import com.ruijie.contract.domain.vo.UndoAssignTelegraphicTransferVO;
import com.ruijie.contract.mapper.IntelContractAssignMapper;
import com.ruijie.contract.mapper.expand.ContractAssignExpandMapper;
import com.ruijie.contract.mapper.expand.ContractInfoExpandMapper;
import com.ruijie.contract.mapper.expand.IntelContractAssignExpandMapper;
import com.ruijie.contract.mapper.expand.TelegraphicTransferInternationExpandMapper;
import com.ruijie.contract.mapper.expand.WaterOrderAttachmentExpandMapper;
import com.ruijie.contract.service.TelegraphicTransferInternationService;
import com.ruijie.contract.util.ContractAPPCode;
import com.ruijie.contract.util.ContractException;

import org.springframework.beans.BeanUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

import lombok.extern.slf4j.Slf4j;

/**
 * @author zrh
 * @date 2020/7/20
 */
@Slf4j
@Service
public class TelegraphicTransferInternationServiceImpl implements TelegraphicTransferInternationService {
    @Autowired
    private TelegraphicTransferInternationExpandMapper telegraphicTransferInternationExpandMapper;
    @Autowired
    private WaterOrderAttachmentExpandMapper waterOrderAttachmentExpandMapper;
    @Autowired
    private ContractAssignExpandMapper contractAssignExpandMapper;
    @Autowired
    private ContractInfoExpandMapper contractInfoExpandMapper;
    @Autowired
    private AbstractContractReceiptHandler abstractContractReceiptHandler;
    @Autowired
    private IntelContractAssignExpandMapper intelContractAssignExpandMapper;

    @Override
    public PageInfo<ListTelegraphicTransferInternationVO> listTelegraphicTransferInternation(TelegraphicTransferInternationVO param) {
        PageHelper.startPage(param.getPageIndex(), param.getPageSize());
        return new PageInfo<ListTelegraphicTransferInternationVO>(telegraphicTransferInternationExpandMapper.listTelegraphicTransferInternation(param));
    }

    @Override
    public TelegraphicTransferInternationDetailVO telegraphicTransferInternationDetail(Integer id) {
        TelegraphicTransferInternationDetailVO telegraphicTransferInternationDetailVO = new TelegraphicTransferInternationDetailVO();
        TelegraphicTransferInternation telegraphicTransferInternation = telegraphicTransferInternationExpandMapper.selectByPrimaryKey(id);
        BeanUtils.copyProperties(telegraphicTransferInternation, telegraphicTransferInternationDetailVO);
        List<WaterOrderAttachment> attachmentList = waterOrderAttachmentExpandMapper.selectByOrderId(Long.valueOf(id));
        telegraphicTransferInternationDetailVO.setAttachmentList(attachmentList);
        return telegraphicTransferInternationDetailVO;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean confirmCollection(TelegraphicConfirmCollectionVO param) {
        //修改汇款状态，实际到账金额，计算手续费，然后清除原有的水单附件，写入新的水单附件
        Integer id = param.getId();
        TelegraphicTransferInternation transferInternation = telegraphicTransferInternationExpandMapper.selectByPrimaryKey(id);
        TelegraphicTransferInternation updateTransfer = new TelegraphicTransferInternation();
        updateTransfer.setId(id);
        updateTransfer.setActualRemittanceAmount(param.getActualRemittanceAmount());
        updateTransfer.setRemittanceState(param.getRemittanceState());
        updateTransfer.setServiceFee(transferInternation.getRemittanceAmount().subtract(param.getActualRemittanceAmount()));
        telegraphicTransferInternationExpandMapper.updateByPrimaryKeySelective(updateTransfer);
        waterOrderAttachmentExpandMapper.deleteIsLcByOrderId(Long.valueOf(id));
        if (param.getLcFiles() != null) {
            param.getLcFiles().stream().forEach((waterOrderAttachment) -> {
                //todo 创建人
                waterOrderAttachment.setOrderId(Long.valueOf(id));
                waterOrderAttachment.setIsDelete(false);
                waterOrderAttachment.setIsLc(true);
                waterOrderAttachment.setCreateTime(new Date());
                waterOrderAttachmentExpandMapper.insertSelective(waterOrderAttachment);
            });
        }
        //抛转来款单处理
        abstractContractReceiptHandler.nextHandler.handlerRequest(transferInternation);
        return true;
    }

    @Override
    public List<WaterOrderAttachment> listWaterOrderAttachment(Long orderId) {
        return waterOrderAttachmentExpandMapper.selectWaterOrderByOrderId(orderId);
    }

    @Override
    public List<ContractAssignVO> listContractAssign(String telegraphicTransferNo) {
        return contractAssignExpandMapper.selectByTelegraphicTransferNum(telegraphicTransferNo);
    }

    @Override
    public PageInfo<ListTelegraphicTransferInternationVO> listAssignTelegraphicTransferInternation(AssignTelegraphicTransferInternationVO param) {
        PageHelper.startPage(param.getPageIndex(), param.getPageSize());
        return new PageInfo<ListTelegraphicTransferInternationVO>(telegraphicTransferInternationExpandMapper.listAssignTelegraphicTransferInternation(param));
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean addTelegraphicTransferInternation(AddTelegraphicTransferInternationVO param) {
        telegraphicTransferInternationExpandMapper.insertSelectiveReturnId(param);
        param.getAttachmentList().stream().forEach((file) -> {
            file.setCreateTime(new Date());
            file.setIsLc(false);
            file.setOrderId(Long.valueOf(param.getId()));
            waterOrderAttachmentExpandMapper.insertSelective(file);
        });
        return true;
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public Boolean assignTelegraphicTransferInternation(AddAssignTelegraphicTransferInternationVO param) {
        param.getAssignDetailList().stream().forEach(assignDetail -> {
            //分配电汇单
            Integer selectRow = telegraphicTransferInternationExpandMapper.assignTelegraphicTransferInternation(assignDetail);
            if (selectRow <= 0) {
                throw new ContractException(ContractAPPCode.ERROR_ASSIGN.getAppCode(), ContractAPPCode.ERROR_ASSIGN.getErrorMessage());
            }
            //分配日志
            IntelContractAssign contractAssign = new IntelContractAssign();
            //todo createBy
            contractAssign.setContractId(param.getContractId());
            contractAssign.setTelegraphicTransferNum(assignDetail.getTelegraphicTransferNo());
            contractAssign.setAssignAmount(assignDetail.getAssignAmount());
            contractAssign.setCreateTime(new Date());
            intelContractAssignExpandMapper.insertSelective(contractAssign);
        });
        //修改合同金额
        BigDecimal assignAll = param.getAssignDetailList().stream().map(AddAssignTelegraphicTransferInternationVO.AssignDetail::getAssignAmount).reduce(BigDecimal.ZERO, BigDecimal::add);
        ContractInfo contractInfo = contractInfoExpandMapper.selectByPrimaryKey(param.getContractId());
        ContractInfo afterContractInfo = new ContractInfo();
        afterContractInfo.setId(param.getContractId());
        afterContractInfo.setAssignAll(contractInfo.getAssignAll().add(assignAll));
        contractInfoExpandMapper.updateByPrimaryKeySelective(afterContractInfo);
        return true;
    }

    @Override
    public Boolean UndoAssignTelegraphicTransferVO(UndoAssignTelegraphicTransferVO param) {
        List<IntelContractAssign> contractAssigns = intelContractAssignExpandMapper.selectByContractId(param.getContractId());
        contractAssigns.stream().forEach(assignDetail -> {
            //撤销电汇单
            Integer selectRow = telegraphicTransferInternationExpandMapper.undoAssignTelegraphicTransferInternation(assignDetail);
            //撤销分配记录
            assignDetail.setIsDelete(true);
            intelContractAssignExpandMapper.updateByPrimaryKeySelective(assignDetail);
        });
        //撤销合同金额
        BigDecimal assignAll = contractAssigns.stream().map(IntelContractAssign::getAssignAmount).reduce(BigDecimal.ZERO, BigDecimal::add);
        ContractInfo contractInfo = contractInfoExpandMapper.selectByPrimaryKey(param.getContractId());
        ContractInfo afterContractInfo = new ContractInfo();
        afterContractInfo.setId(param.getContractId());
        afterContractInfo.setAssignAll(contractInfo.getAssignAll().subtract(assignAll));
        contractInfoExpandMapper.updateByPrimaryKeySelective(afterContractInfo);
        return true;
    }

}

```
# 不合理的代码
## 通过类对象引用类静态变量
```code
/**
 * 系统日志：切面处理类
 */
@Slf4j
@Aspect
@Component
public class SysLogAspect {
    @Autowired
    SysRequestLogService sysRequestLogService;
    //在注解的位置切入代码
    @Pointcut(value="@annotation(AnnotationLog)")
    public void logPoinCut() {
    }
    //切面 配置通知
    @Before(value="logPoinCut()", argNames = "joinPoint")
    public void saveSysLog(JoinPoint joinPoint) {
        //保存日志
        SysRequestLog operlog = new SysRequestLog();
        try {
            //生成主键
            IdWorker worker2 = new IdWorker(1);
        }
    }
}

public class IdWorker {

    private final long workerId;
    private final static long twepoch = 1288834974657L;
    private long sequence = 0L;
    private final static long workerIdBits = 4L;
    public final static long maxWorkerId = -1L ^ -1L << workerIdBits;
    private final static long sequenceBits = 10L;
    private final static long workerIdShift = sequenceBits;
    private final static long timestampLeftShift = sequenceBits + workerIdBits;
    public final static long sequenceMask = -1L ^ -1L << sequenceBits;
    private long lastTimestamp = -1L;
    public IdWorker(final long workerId) {
        super();
        if (workerId > this.maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format(
                    "worker Id can't be greater than %d or less than 0",
                    this.maxWorkerId));
        }
        this.workerId = workerId;
    }
}
```
## 应以常量作为equals的调用方，如果对象为空调用equals会触发空指针
```code
@Slf4j
@Component
public class PostContractERPService implements SimpleJob{
    @Override
    public void execute(ShardingContext shardingContext) {
        try {
            if(returnFlag.equals("F")){
                //同步post信息失败，发送通知邮件
            }
        } catch (Exception e) {
            log.error("抛单信息同步到ECP失败,postmain信息:{}", JSON.toJSONString(postMain));
        }
    }
}
``` 
## 线程池参数设置,最好自定义线程名字，方便排查问题，同样根据业务来看阻塞队列过大，最大线程可能一直无法发挥用处
```code
@Slf4j
@RestController
@RequestMapping("/contractInfo")
public class ContractInfoController {
    ThreadPoolExecutor executor = new ThreadPoolExecutor(2,5,30, TimeUnit.MINUTES,new ArrayBlockingQueue(200));
```
## 常量名全部大写，而且最好能表达常量的含义，不需要驼峰，也不需要刻意短的命名
```code
public class ContractAPPCode extends AbstracAppCode {
    public static final ContractAPPCode ERROR_EmptyParameter = new ContractAPPCode("参数为空!","7100002");
    public static final ContractAPPCode ERROR_PARAM_AGENT_NAME = new ContractAPPCode("代理商名称为空!","7100003");
    public static final ContractAPPCode ERROR_PARAM_CONTRACTASSIGN_ERR = new ContractAPPCode("合同金额分配异常！","7100004");
    public static final ContractAPPCode ERROR_PARAM_ASSIGN_ERR = new ContractAPPCode("生成收款单异常！","7100005");
    public static final ContractAPPCode ERROR_USERId = new ContractAPPCode("操作人信息为空！","7100006");
    public static final ContractAPPCode ERROR_Comparative_Amount = new ContractAPPCode("申请退款金额不能超过可用余额！","7100007");
    public static final ContractAPPCode ERROR_ISCurrentApprover = new ContractAPPCode("您不是当前审批人!","7100008");

    public static final ContractAPPCode ERROR_CONTRACT_NOTEXIS = new ContractAPPCode("合同中心不存在，该合同信息，请联系开发人员！","7100009");
    public static final ContractAPPCode ERROR_CONTRACT_NOTDiscard = new ContractAPPCode("合同中心执行更新废弃状态失败！","7100010");
    public static final ContractAPPCode ERROR_CONTRACT_EXISDiscard = new ContractAPPCode("该合同已经被废弃过！","7100011");

```
## 常量最好不要直接出现在代码里
```code
//针对 总条数 大于 1000000 目前忽略
            if (jsonObject.get("status").equals(200) && count > 0
                    && !Strings.isNullOrEmpty(jsonObject.get("data").toString())

```
## 取反逻辑不利于快速理解代码,所有!可以做到的，不用!一样可以做到
```code
        if(!(i1>0)){
            String msg = "待办合同款项分配-插入分配记录失败！";
            throw new ContractException(ContractAPPCode.ERROR_PARAM_CONTRACTASSIGN_ERR.getAppCode(),msg);
        }
```
# 个人想法

# 如何提高代码质量
## 不要轻易复制逻辑代码
避免有的地方有改动，要同步到其他的地方，如果有遗漏，就会产生问题，最好重构它
## 排期内最好预留时间对所写代码重构设计
一般为五分之一的时间，好的代码设计不仅不不会浪费这五分之一的时间，还将对以后产生深远的影响，重点在提高可维护性和可读性
## 代码量
有足够多的代码量才会对各种各样的规范有自己的理解，总结，反思
# 分享自己的代码
..........
.........
.......
