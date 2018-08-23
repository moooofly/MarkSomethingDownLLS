# shell 脚本中颜色输出问题

> 写在开始：
>
> - 若使用 `#!/bin/sh` ，由于其链接到 `dash` 上，因此在使用 echo 输出时不需要指定 -e 选项；
> - 若使用 `#!/bin/bash` ，则必须指定 -e 选项才行；

输出示例

```
echo -e "\e[1;34mThis is a blue text.\e[0m"
```

控制开关

- \e[`attribute code`;`text color code`;`background color code`m
- \e[0m

> 经确认，`attribute code`、`text color code` 和 `background color code` 的顺序没有严格要求，因为数值范围不同，不会导致错误；

| Attribute codes | 英文 | 中文 |
| -- | -- | -- |
| 00 | none | 重新设置属性到缺省设置 | 
| 01 | bold | 设置粗体 | 
| 02 | | 设置一半亮度 |
| 03 | | 设置斜体 |
| 04 | underscore | 设置下划线 | 
| 05 | blink | 设置闪烁 | 
| 07 | reverse | 反白显示 | 
| 08 | concealed | 不可见/消隐 | 

| Text color codes | 英文 | 中文 |
| -- | -- | -- |
| 30 | black | 黑色 |
| 31 | red | 红色 |
| 32 | green | 绿色 |
| 33 | yellow | 黄色 |
| 34 | blue | 蓝色 |
| 35 | magenta | 紫红色 |
| 36 | cyan | 青蓝色 |
| 37 | white | 白色 |
| 38 | | 在缺省的前景颜色上设置下划线 |
| 39 | | 在缺省的前景颜色上关闭下划线 |

| Background color codes | 英文 | 中文 |
| -- | -- | -- |
| 40 | black | 黑色 |
| 41 | red | 红色 |
| 42 | green | 绿色 |
| 43 | yellow | 黄色 |
| 44 | blue | 蓝色 |
| 45 | magenta | 紫红色 |
| 46 | cyan | 青蓝色 |
| 47 | white | 白色 |


可以输出全部颜色的脚本（略微调整）

```
#/bin/bash

for STYLE in 0 1 2 3 4 5 6 7; do
  for FG in 30 31 32 33 34 35 36 37; do
    for BG in 40 41 42 43 44 45 46 47; do
      CTRL="\033[${STYLE};${FG};${BG}m"
      END="\033[0m"
      echo "${CTRL}moooofly${END} <--> ${STYLE};${FG};${BG}"
    done
    echo
  done
  echo
done
# Reset
echo "\033[0m"
```

更实际的一个脚本

```
#!/bin/bash

# NOTE:
# 若使用 #!/bin/sh ，由于其链接到 dash 上，因此使用 echo 输出时不需要指定 -e 选项
# 若使用 #!/bin/bash ，则必须指定 -e 选项才行

echo -e "\033[47;30;5m david use echo say \033[0m Hello World"

echo -e "\033[0m none \033[0m"
echo -e "\033[30m black \033[0m"
echo -e "\033[1;30m dark_gray \033[0m"
echo -e "\033[0;34m blue \033[0m"
echo -e "\033[1;34m light_blue \033[0m"
echo -e "\033[0;32m green \033[0m"
echo -e "\033[1;32m light_green \033[0m"
echo -e "\033[0;36m cyan \033[0m"
echo -e "\033[1;36m light_cyan \033[0m"

echo -e "\033[0;31m red \033[0m"
echo -e "\033[1;31m light_red \033[0m"
echo -e "\033[0;35m purple \033[0m"
echo -e "\033[1;35m light_purple \033[0m"
echo -e "\033[0;33m brown \033[0m"
echo -e "\033[1;33m yellow \033[0m"
echo -e "\033[0;37m light_gray \033[0m"
echo -e "\033[1;37m white \033[0m"
echo -e "\033[0m none \033[0m"

echo -e "\033[40;37m 黑底白字 \033[0m"
echo -e "\033[41;30m 红底黑字 \033[0m"
echo -e "\033[41;30;1m 红底加粗黑字 \033[0m"
echo -e "\033[42;34m 绿底蓝字 \033[0m"
echo -e "\033[42;34;1m 绿底加粗蓝字 \033[0m"
echo -e "\033[42;30;1m 绿底加粗黑字 \033[0m"
echo -e "\033[43;34m 黄底蓝字 \033[0m"
echo -e "\033[44;30m 蓝底黑字 \033[0m"
echo -e "\033[44;30;1m 蓝底加粗黑字 \033[0m"
echo -e "\033[45;30m 紫底黑字 \033[0m"
echo -e "\033[46;30m 天蓝底黑字 \033[0m"
echo -e "\033[46;30;1m 天蓝底加粗黑字 \033[0m"
echo -e "\033[47;34m 白底蓝字 \033[0m"
echo -e "\033[47;30m 白底黑字 \033[0m"
echo -e "\033[47;30;1m 白底加粗黑字 \033[0m"
echo -e "\033[4;31m 下划线红字 \033[0m"
echo -e "\033[5;31m 红字在闪烁 \033[0m"
echo -e "\033[8m 消隐 \033[0m"

echo "---------"

RED_COLOR='\033[1;31m'
YELOW_COLOR='\033[1;33m'
BLUE_COLOR='\033[1;34m'
RESET='\033[0m'

echo -e "${RED_COLOR}===david say red color===${RESET}"
echo -e "${YELOW_COLOR}===david say yelow color===${RESET}"
echo -e "${BLUE_COLOR}===david say green color===${RESET}"

echo "---------"

ESC="

RED_COLOR="${ESC}[31m"
YELOW_COLOR="${ESC}[33m"
BLUE_COLOR="${ESC}[34m"
RESET="${ESC}[0m"

echo -e "${RED_COLOR}===david say red color===${RESET}"
echo -e "${YELOW_COLOR}===david say yelow color===${RESET}"
echo -e "${BLUE_COLOR}===david say green color===${RESET}"
```

----------


参考：

- [Bash: Using Colors](http://webhome.csc.uvic.ca/~sae/seng265/fall04/tips/s265s047-tips/bash-using-colors.html)
- [shell脚本输出输出带颜色内容](http://blog.csdn.net/David_Dai_1108/article/details/70478826)
- [Bash Shell怎么打印各种颜色](https://segmentfault.com/q/1010000000122806)





