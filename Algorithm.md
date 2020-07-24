# Algorithm

## Sort

### 插入排序

```java

int[] data = {1, 7, 5, 9, 2, 4, 5};

        for (int i = 1; i < data.length; i++) {
            int num = data[i];
            if (num < data[i - 1]) {
                for (int j = i - 1; j >= 0; j--) {
                    if (num < data[j]) {
                        data[j + 1] = data[j];
                    } else {
                        data[j+1] = num;
                        break;
                    }
                }
            }
        }

        System.out.println(Arrays.toString(data));

```

