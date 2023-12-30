# shell_public
public 오픈된 Shell

## aws-cloud9
cluod9 의 volume을 50 G로 변경해주는 resize.sh 
```
curl -fsSL https://raw.githubusercontent.com/spc1jh/shell_public/main/aws-cloud9/resize.sh | bash
df -h
```

용량을 변경하기 위해서는 다음과 같이 하면 됩니다. 
```
curl -fsSL https://raw.githubusercontent.com/spc1jh/shell_public/main/aws-cloud9/resize.sh | bash -s 100
df -h
```
