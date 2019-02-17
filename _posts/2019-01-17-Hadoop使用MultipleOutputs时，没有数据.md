---
layout: post
title: Hadoop使用MultipleOutputs时，没有数据
date: 2019-02-17
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- 错误处理
tags:
- 错误处理
---
在这么一个场景中，我们用到了**MultipleOutputs**。

我们在进行统计之后，想要将其按照Key输出到不同的目录。

在使用**MultipleOutputs**来做这件事的时候，就发现，目录能够被正确创建，但是对应的文件却老是没有内容。

读了一遍**MultipleOutputs**的源码，发现应该没有问题啊。

最后，请教我们组的Leader，才知道问题原来是出在了没有调用两个方法。

在我们用Context来输出的时候，我们直接调用**context.write(key, value)**就能将结果输出到正确的目录，但是调用**multipleOutputs.write(key,value)**时，并不是这样。

它会先将你要写的这些内容，保存到一个缓冲区中，只有当你调用**multipleOutputs.close()和Reducer的super.cleanup(context)**时，才正确。

示例代码如下：

~~~~
package com.company.reducer;

import com.company.datastructure.SuffixCompareText;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.output.MultipleOutputs;

import java.io.IOException;

public class SuffixWordCountReducerWithOutputToCustomizedLocation extends Reducer<SuffixCompareText, IntWritable, Text, Text>{

    private int total;
    private int unique;
    private String suffixOfPreviousPartition = null;
    private Text outputKey = new Text();
    private Text outputValue = new Text();

    private MultipleOutputs<Text, Text> mos;

    @Override
    public void setup(Context context) {
        mos = new MultipleOutputs(context);
    }

    @Override
    public void reduce(SuffixCompareText key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
        String keyString = key.toString();
        String suffixOfCurrentKey = keyString.substring(keyString.length() - 1);
        if (!suffixOfCurrentKey.equals(suffixOfPreviousPartition)) {
            if (suffixOfPreviousPartition != null) {
                outputKey.set(suffixOfPreviousPartition);
                outputValue.set(Integer.toString(total) + " " + Integer.toString(unique));
                mos.write(outputKey, outputValue, generateFileName());
            }
            suffixOfPreviousPartition = suffixOfCurrentKey;
            total = 0;
            unique = 0;
        }

        for (IntWritable value : values) {
            total += value.get();
        }
        unique++;

    }

    @Override
    public void cleanup(Context context) throws IOException, InterruptedException {
        if (suffixOfPreviousPartition != null) {
            outputKey.set(suffixOfPreviousPartition);
            outputValue.set(Integer.toString(total) + " " + Integer.toString(unique));
            mos.write(outputKey, outputValue, generateFileName());
        }
        // You should call these two lines to flush the data to hdfs because the data is saved in buffer when you call write.
        mos.close();
        super.cleanup(context);
    }

    private String generateFileName() {
        return outputKey + "/" + outputKey;
    }

}
~~~~
