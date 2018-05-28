### 顺德ems文书生成功能分析

```java
//统一文书生成功能
@RequestMapping("commonCreateWs")
@ResponseBody
public String commonCreateWs(CommonCreateWs info, HttpServletRequest request, HttpServletResponse response, @RequestParam(value = "formKey", required = true) String formKey) {
    HttpSession session = request.getSession();
    boolean firstCommite = false;
    try {
        String form = (String) session.getAttribute("formKey");//这行代码报异常 ,说明没有这个属性。
        if (formKey.equals(form)) {
            firstCommite = true;//判断为第一次提交。
            session.removeAttribute("formKey");//清空session的这个属性
        }else {
            //不是的一次提交
        }
    } catch (Exception e) {
        e.printStackTrace();
        //什么都不要做
        logger.info("=====================================================哈哈哈===================================");
    }
	// 提交
    if (firstCommite) {
        Map<String, Object> result = zxAjxxService.commonCreateWs(info); // 统一生成文书
        String wsKey = (String) result.get("wsKey");
        switch (wsKey) {
            case ZxWsKey.ZXTZS:
                return (String) result.get("data");
            case ZxWsKey.PMWS:
                downLoadZip(info, response, result);
                return "";
            default:
                return "";
        }
    }
    return "生成成功！";
}
```

```java
@Transactional(readOnly = false)
public Map<String, Object> commonCreateWs(CommonCreateWs info) {
    Map<String, Object> resultMap = Maps.newHashMap();
   //执行通知书文书生成
        switch (info.getWsKey()) {
            case ZxWsKey.ZXTZS://执行通知书
                resultMap = createZxtzs(info);
                break;

            case ZxWsKey.PMWS://拍卖文书
                resultMap.put("data",createPmws(info));
                break;

            case ZxWsKey.SXCJ://失信惩戒文书
                 resultMap.put("data",createSxcj(info));
                break;

            case ZxWsKey.QZCD://强制裁定文书
                resultMap.put("data",createQzcd(info));
                break;

            case ZxWsKey.JCCJ://拘传惩戒
                resultMap.put("data",createJccj(info));
                break;

            case ZxWsKey.JLCJ://拘留惩戒
                resultMap.put("data",createZlcj(info));
                break;
            default:
        }
        resultMap.put("wsKey", info.getWsKey());
        resultMap.put("code", "1");
        return resultMap;
    return resultMap;
}
```

```java
@Transactional(readOnly = false)
private Map<String, Object> createZxtzs(CommonCreateWs info) {

    String wsKey = info.getWsKey();
    String arrivedTime = info.getArrivedTime();
    String sxStart = info.getSxqxStart();
    String sxEnd = info.getSxqxEnd();
    String sxdyt = info.getSxdyt();
    List<String> sxyyList = info.getSxyy();
    String sxyy = "";
    if (sxyyList != null && sxyyList.size() > 0) {
        for (String yy : sxyyList) {
            sxyy += yy;
        }
    }
    String ahdm = info.getAhdm();
    List<String> mbs = info.getMbs();
    List<EmsMb> emsMbs = new ArrayList<EmsMb>();
    for (String id : mbs) {
        Integer mbId = Integer.parseInt(id);
        EmsMb emsMb = emsMbDao.selectByPrimaryKey(mbId);
        emsMbs.add(emsMb);
    }
    //SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm");
    ZxAjxx zxAjxx = dao.getByAhdm(ahdm);
    //--承办人和书记员联系方式
    Map<String, String> lxMap = cbrAndSjyLXFS(zxAjxx, true, false, true, false);
    ZxAjxxFjb zxAjxxfjb = zxAjxxFjbDao.get(ahdm);
    Boolean hasCp = false;//有传票
    Boolean hasZdxw = false;//执行通知书_责令履行指定行为
    for (EmsMb mb : emsMbs) {
        String mbName = mb.getWs();
        if (mbName.contains("传票")) {
            hasCp = true;
        }
        if (mbName.contains("执行通知书_责令履行指定行为")) {
            hasZdxw = true;
        }
    }
    // 1 所有需要替换的标签
    List<String> labelList = getBaseLabelList();
    labelList.add("【应到时间】");
    labelList.add("【第一条款项】");//1~6
    labelList.add("【第二条第一款】");//有纳入期限则有
    labelList.add("【失信原因】");
    labelList.add("【纳入期限信息】"); //有纳入期限则有 （纳入期限为×年，自××××年×月×日至××××年×月×日）
    labelList.add("【承办人电话】");
    labelList.add("【书记员电话】");
    labelList.add("【承办人执行工作微信号二维码】");
    labelList.add("【当事人】");
    labelList.add("【暂住地址】");
    labelList.add("【判项条款】");

    //2 标签对应的值。
    Map<String, String> replaceMap = baseReplaceMap(info.getAhdm());
    //replaceMap.("【承办人执行工作微信号二维码】","");
    replaceMap.put("【应到时间】", arrivedTime == null || "".equals(arrivedTime) ? "" : changeArrivedTime(arrivedTime));
    replaceMap.put("【第一条款项】", sxdyt == null ? "" : sxdyt);
    replaceMap.put("【第二条第一款】", sxStart != null && !"".equals(sxStart) && sxEnd != null && !"".equals(sxEnd) ? "第二条第一款、" : "");
    replaceMap.put("【失信原因】", sxyy.length() == 0 ? "" : sxyy.substring(1));
    if (sxStart != null && !"".equals(sxStart) && sxEnd != null && !"".equals(sxEnd)) {
        SimpleDateFormat format1 = new SimpleDateFormat("yyyy-MM-dd");
        String start = changeSxTime(sxStart);
        String end = changeSxTime(sxEnd);

        Calendar c1 = Calendar.getInstance();
        Calendar c2 = Calendar.getInstance();
        try {
            c1.setTime(format1.parse(sxStart));
            c2.setTime(format1.parse(sxEnd));
        } catch (ParseException e) {
            e.printStackTrace();
            logger.error("失信限制日期解析失败！");
            throwMessage("失信限制日期解析失败！");
        }
        int year1 = c1.get(Calendar.YEAR);
        int year2 = c2.get(Calendar.YEAR);
        replaceMap.put("【纳入期限信息】", "，纳入期限为" + (year2 - year1) + "年，自" + start + "至" + end);
    } else {
        replaceMap.put("【纳入期限信息】", "");
    }
    replaceMap.put("【承办人电话】", lxMap.get("cbrPhone"));
    replaceMap.put("【书记员电话】", lxMap.get("sjyPhone"));

    if (hasZdxw) {
        if (StringUtils.isNotBlank(zxAjxxfjb.getTk())) {
            replaceMap.put("【判项条款】", zxAjxxfjb.getTk());
        } else {
            throwMessage("判项条款内容不存在，请您进行维护！");
        }
    }

    //3 需要生成文书的当事人
    List<ZxDsr> dsrs = zxDsrDao.findListByAhdm(ahdm);

    //4 需要替换的模板  按照申请执行人，被执行分类
    Map<String, List<EmsMb>> ssdwMbs = ssdwMbList(emsMbs);

    //5 word文件生成 存入ems_wsxx_flws表当中,在ems_wsxx表 ems_dsr信息表都记录。
    List<EmsMb> needMbs = new ArrayList<EmsMb>();
    for (ZxDsr dsr : dsrs) {
        String emsLxs = "";
        String wsmc = dsr.getMc() + "送达材料";
        EmsWsxx emsWsxx = getPdfEmsWsxx(wsmc, zxAjxx);
        emsWsxx.setWsKey(ZxWsKey.ZXTZS);
        //执行申请人
        if ("09_05036-1".equals(dsr.getSsdw1())) {
            needMbs = ssdwMbs.get("sqrMbs");
            replaceMap.put("【当事人】", dsr.getMc());
            //被执行人
        } else if ("09_05036-2".equals(dsr.getSsdw1())) {
            needMbs = ssdwMbs.get("bzxrMbs");
            replaceMap.put("【当事人】", dsr.getMc());
            if (hasCp) {
                replaceDz(replaceMap, dsr);
            }
        }
        for (EmsMb mb : needMbs) {
            String path = mb.getWsmblj();
            if (mb.getEmsLx() != null && !"".equals(mb.getEmsLx())) {
                emsLxs += (mb.getEmsLx() + ",");
            }
            //替换标签--入库flws--逻辑
            wsReplaceAndRecordInTb(mb, emsWsxx, replaceMap, labelList);
        }

        if (emsLxs.length() > 0) {
            emsWsxx.setWslx(emsLxs.substring(0, emsLxs.length() - 1));
        }
        //防止没有对应模板也给 dsr 插入信息。
        if (Collections3.checkListNotEmpty(needMbs)) {
            emsWsxxDao.insert(emsWsxx);
            insertEmsDsr(emsWsxx, dsr);
        }

        // 5 为被执行人生成传票--绑定传票存根的信息
        if ("09_05036-2".equals(dsr.getSsdw1())) {
            List<EmsMb> mbList = ssdwMbs.get("bzxrMbs");
            if (Collections3.checkListNotEmpty(mbList)) {
                //boolean flag = false;
                for (EmsMb emsMb : mbList) {
                    if ("传票".equals(emsMb.getWs())) {
                        String cg = dsr.getMc() + "传票（存根）)";
                        EmsWsxx wsxx = getPdfEmsWsxx(cg, zxAjxx);
                        EmsMb mb = new EmsMb();
                        mb.setWs("传票（存根）");
                        mb.setWsmblj(emsMb.getWsmblj().substring(0, emsMb.getWsmblj().lastIndexOf(".")) + "（存根）.doc");
                        mb.setFyid(emsMb.getFyid());
                        mb.setGz(emsMb.getGz());
                        mb.setGzwz(emsMb.getGzwz());
                        wsReplaceAndRecordInTb(mb, wsxx, replaceMap, labelList);
                        emsWsxxDao.insert(wsxx);
                        //存入关系表当中
                        Map<String, String> infMap = Maps.newHashMap();
                        infMap.put("id", IdGen.uuid());
                        infMap.put("wsxx_a_id", emsWsxx.getId());
                        infMap.put("wsxx_b_id", wsxx.getId());
                        emsWsxxDao.insertReference(infMap);
                    }
                }
            }
        }
    }

    Map<String, Object> infoMap = Maps.newHashMap();
    infoMap.put("code", "1");
    infoMap.put("data", "生成文书成功");
    return infoMap;
}
```

思路：

* 每种文书都有自己的替换标签，目前需要的是先判断生成文书类型，然后根据类型手动添加标签list，然后执行替换方法，将list含有的标签替换

改进思路：

* 先获取生成文书类型，然后读取该模板，根据正则匹配标签，然后获取标签对应的值，再进行替换。
* 获取模板中的标签



每种文书生成方法：

* zxtzs执行通知书

  ```Java
  @Transactional(readOnly = false)
  private Map<String, Object> createZxtzs(CommonCreateWs info) {
      String wsKey = info.getWsKey();
      String arrivedTime = info.getArrivedTime();
      String ahdm = info.getAhdm();
      List<String> mbs = info.getMbs(); // 模板表中的Id
      List<EmsMb> emsMbs = new ArrayList<EmsMb>();
      // 取得模板id对应的ems模板
      for (String id : mbs) {
          Integer mbId = Integer.parseInt(id);
          EmsMb emsMb = emsMbDao.selectByPrimaryKey(mbId);
          emsMbs.add(emsMb);
      }
      ZxAjxx zxAjxx = dao.getByAhdm(ahdm);
      ZxAjxxFjb zxAjxxfjb = zxAjxxFjbDao.get(ahdm);
      Boolean hasCp = false;//有传票
      Boolean hasZdxw = false;//执行通知书_责令履行指定行为
      for (EmsMb mb : emsMbs) {
          String mbName = mb.getWs();
          if (mbName.contains("传票")) {
              hasCp = true;
          }
          if (mbName.contains("执行通知书_责令履行指定行为")) {
              hasZdxw = true;
          }
      }
      // 1 所有需要替换的标签
      List<String> labelList = getBaseLabelList();
      labelList.add("【应到时间】");
      labelList.add("【承办人电话】");
      labelList.add("【书记员电话】");
      labelList.add("【承办人执行工作微信号二维码】");
      labelList.add("【当事人】");
      labelList.add("【暂住地址】");
      labelList.add("【判项条款】");
  
      //2 标签对应的值。
      Map<String, String> replaceMap = baseReplaceMap(info.getAhdm(),true);
      replaceMap.put("【应到时间】", arrivedTime == null || "".equals(arrivedTime) ? "" : changeArrivedTime(arrivedTime));
  
      if (hasZdxw) {
          if (StringUtils.isNotBlank(zxAjxxfjb.getTk())) {
              replaceMap.put("【判项条款】", zxAjxxfjb.getTk());
          } else {
              throwMessage("判项条款内容不存在，请您进行维护！");
          }
      }
      //3 需要生成文书的当事人
      List<ZxDsr> dsrs = zxDsrDao.findListByAhdm(ahdm);
  
      //4 需要替换的模板  按照申请执行人，被执行分类
      Map<String, List<EmsMb>> ssdwMbs = ssdwMbList(emsMbs);
  
      //5 word文件生成 存入ems_wsxx_flws表当中,在ems_wsxx表 ems_dsr信息表都记录。
      List<EmsMb> needMbs = new ArrayList<EmsMb>();
      for (ZxDsr dsr : dsrs) {
          String emsLxs = "";
          String wsmc = dsr.getMc() + "送达材料";
          EmsWsxx emsWsxx = getPdfEmsWsxx(wsmc, zxAjxx);
          emsWsxx.setWsKey(ZxWsKey.ZXTZS);
          //执行申请人
          if ("09_05036-1".equals(dsr.getSsdw1())) {
              needMbs = ssdwMbs.get("sqrMbs");
              replaceMap.put("【当事人】", dsr.getMc());
          //被执行人
          } else if ("09_05036-2".equals(dsr.getSsdw1())) {
              needMbs = ssdwMbs.get("bzxrMbs");
              replaceMap.put("【当事人】", dsr.getMc());
              if (hasCp) {
                  replaceDz(replaceMap, dsr);
              }
          }
          for (EmsMb mb : needMbs) {
              String path = mb.getWsmblj();
              if (mb.getEmsLx() != null && !"".equals(mb.getEmsLx())) {
                  emsLxs += (mb.getEmsLx() + ",");
              }
              //替换标签--入库flws--逻辑
              wsReplaceAndRecordInTb(mb, emsWsxx, replaceMap, labelList);
          }
  
          if (emsLxs.length() > 0) {
              emsWsxx.setWslx(emsLxs.substring(0, emsLxs.length() - 1));
          }
          //防止没有对应模板也给 dsr 插入信息。
          if (Collections3.checkListNotEmpty(needMbs)) {
              emsWsxxDao.insert(emsWsxx);
              insertEmsDsr(emsWsxx, dsr);
          }
  
          // 5 为被执行人生成传票--绑定传票存根的信息
          if ("09_05036-2".equals(dsr.getSsdw1())) {
              List<EmsMb> mbList = ssdwMbs.get("bzxrMbs");
              if (Collections3.checkListNotEmpty(mbList)) {
                  //boolean flag = false;
                  for (EmsMb emsMb : mbList) {
                      if ("传票".equals(emsMb.getWs())) {
                          String cg = dsr.getMc() + "传票（存根）)";
                          EmsWsxx wsxx = getPdfEmsWsxx(cg, zxAjxx);
                          EmsMb mb = new EmsMb();
                          mb.setWs("传票（存根）");
                          mb.setWsmblj(emsMb.getWsmblj().substring(0, emsMb.getWsmblj().lastIndexOf(".")) + "（存根）.doc");
                          mb.setFyid(emsMb.getFyid());
                          mb.setGz(emsMb.getGz());
                          mb.setGzwz(emsMb.getGzwz());
                          wsReplaceAndRecordInTb(mb, wsxx, replaceMap, labelList);
                          emsWsxxDao.insert(wsxx);
                          //存入关系表当中
                          Map<String, String> infMap = Maps.newHashMap();
                          infMap.put("id", IdGen.uuid());
                          infMap.put("wsxx_a_id", emsWsxx.getId());
                          infMap.put("wsxx_b_id", wsxx.getId());
                          emsWsxxDao.insertReference(infMap);
                      }
                  }
              }
          }
      }
  
      Map<String, Object> infoMap = Maps.newHashMap();
      infoMap.put("code", "1");
      infoMap.put("data", "生成文书成功");
      return infoMap;
  }
  ```

先得到所有需要替换的模板，然后扫描标签，扫描到一个就执行对应的替换方法

```java
private byte[] createWsTest(CommonCreat)
```

今天主要是研究生成文书的替换标签功能：

1. 目前是每次生产通用的resultMap获取通用的标签值，不同的文书特定的标签在其对应的create方法中，通过add来进行添加
2. 目前改进是通过实现通用的方法，直接加载需要生成的模板，利用扫描其中的标签，存入labelList中，再将该labelList传入得到resultMap的方法中，生成和labelList一一对应的value，不必像之前需要手动add标签。

遇到的问题：

1. 一开始的想法是直接扫描到一个标签就调用该标签的对应替换方法，发现会造成定位错误的问题，因为替换会使一开始匹配到的位置发生变化，因此选择将匹配到的label存到list中，再统一操作

进度：

* 目前实现了基础label的替换，特定文书label的还有待添加