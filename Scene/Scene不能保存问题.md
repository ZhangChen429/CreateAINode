# Scnen不能正确修改与保存

## 根源问题

- Scene保存涉及一些特殊的配置文件的修正需要将计算机名称配置到相应的Config文件中；其中包含特定的格式信息，格式规范来源于源代码(2077\CDPR2077\dev\src\common\engine\src\ruidGenerator.cpp 216-234 |  240-267)

## 解决方法

- 修改配置文件2077\CDPR2077\engine\config\dev\general.ini
- 修改配置文件2077\CDPR2077\r6\config\dev\general.ini
- 修改配置文件2077\CDPR2077\r6\config\user.ini


2077\CDPR2077\engine\config\dev\general.ini
2077\CDPR2077\r6\config\dev\general.ini
2077\CDPR2077\r6\config\user.ini
● Write(E:\SoftApp\Sy2077\2077\2077\CDPR2077\engine\config\dev\general.ini)
  ⎿  Updated ..\..\engine\config\dev\general.ini with 6 additions and 4 removals
    //   1 -  [General]
    //   2 -  ; Development environment configuration
    //   3 -  ; Map your hostname to a valid host ID (1-999)
    //   4 -  cvRUIDGeneratorFakeHostNameMapping = "DESK-W-617=100"
       1 +  [Backend/RUIDGenerator]
       2 +  ; Development environment configuration
       3 +  ; Map your real hostname to a CDPR workstation name
       4 +  ; Format: "RealHostName:CDPRS-ID###" or "RealHostName:CDPRS-BUILD###"
       5 +  ; CDPRS-ID: range 500-3500, CDPRS-BUILD: range 1-200
       6 +  FakeHostNameMapping = "DESK-W-617:CDPRS-BUILD100"

● Write(E:\SoftApp\Sy2077\2077\2077\CDPR2077\r6\config\dev\general.ini)
  ⎿  Updated ..\config\dev\general.ini with 5 additions and 4 removals
     //  1 -  [General]
     //  2 -  ; Development environment configuration
     //  3 -  ; Map your hostname to a valid host ID (1-999)
     //  4 -  cvRUIDGeneratorFakeHostNameMapping = "DESK-W-617=100"
       1 +  [Backend/RUIDGenerator]
       2 +  ; Development environment configuration
       3 +  ; Map your real hostname to a CDPR workstation name
       4 +  ; Format: "RealHostName:CDPRS-ID###" or "RealHostName:CDPRS-BUILD###"
       5 +  FakeHostNameMapping = "DESK-W-617:CDPRS-BUILD100"

● Update(E:\SoftApp\Sy2077\2077\2077\CDPR2077\r6\config\user.ini)
  ⎿  Updated ..\config\user.ini with 2 additions and 1 removal
       78    PadRotationExitTime=0.500000
       79    [Engine]
       80    LimitFPS=10
    //   81 -  cvRUIDGeneratorFakeHostNameMapping="DESK-W-617=100"
       81 +  [Backend/RUIDGenerator]
       82 +  FakeHostNameMapping="DESK-W-617:CDPRS-BUILD100"
       83    [editor/FilterPresetFile/Animgrapheditor]
       84    Value="default.json"
       85    Type=true
