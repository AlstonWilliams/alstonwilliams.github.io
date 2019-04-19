---
layout: post
title: Spark TableInput读HBase是通过直接读HFile么？
date: 2019-04-19
author: AlstonWilliams
header-img: img/post-bg-2015.jpg
catalog: true
categories:
- Spark
tags:
- Spark
---

最近在做一件事情，需要读HBase中全表的数据进行处理，本来是通过一定的处理，然后得到需要读取的record，然后直接Get。但是最后发现这种方式中间总会遇到问题，即跟ZooKeeper的connection expired并且重试了好多次都失败，导致整个任务失败。

后来经过同事提醒，可以直接通过Spark提供的`newAPIHadoopRDD`来全表读HBase，使用下来发现没有遇到上述的ZooKeeper的问题。于是猜测通过这种方式是直接读的HFile，所以没什么问题。但是呢，后来在客户的环境发现，即Scan超时，于是就想看看到底是怎么读HBase的。

最后读过代码发现，不是通过直接读HFile来读的，而是通过Scan的方式来读的。相关代码在`TableRecordReaderImpl`中:
~~~
/**
 * Restart from survivable exceptions by creating a new scanner.
 *
 * @param firstRow  The first row to start at.
 * @throws IOException When restarting fails.
 */
public void restart(byte[] firstRow) throws IOException {
  currentScan = new Scan(scan);
  currentScan.setStartRow(firstRow);
  currentScan.setScanMetricsEnabled(true);
  if (this.scanner != null) {
    if (logScannerActivity) {
      LOG.info("Closing the previously opened scanner object.");
    }
    this.scanner.close();
  }
  this.scanner = this.htable.getScanner(currentScan);
  if (logScannerActivity) {
    LOG.info("Current scan=" + currentScan.toString());
    timestamp = System.currentTimeMillis();
    rowcount = 0;
  }
}

/**
  * Positions the record reader to the next record.
  *
  * @return <code>true</code> if there was another record.
  * @throws IOException When reading the record failed.
  * @throws InterruptedException When the job was aborted.
  */
 public boolean nextKeyValue() throws IOException, InterruptedException {
   if (key == null) key = new ImmutableBytesWritable();
   if (value == null) value = new Result();
   try {
     try {
       value = this.scanner.next();
       if (value != null && value.isStale()) numStale++;
       if (logScannerActivity) {
         rowcount ++;
         if (rowcount >= logPerRowCount) {
           long now = System.currentTimeMillis();
           LOG.info("Mapper took " + (now-timestamp)
             + "ms to process " + rowcount + " rows");
           timestamp = now;
           rowcount = 0;
         }
       }
     } catch (IOException e) {
       // do not retry if the exception tells us not to do so
       if (e instanceof DoNotRetryIOException) {
         throw e;
       }
       // try to handle all other IOExceptions by restarting
       // the scanner, if the second call fails, it will be rethrown
       LOG.info("recovered from " + StringUtils.stringifyException(e));
       if (lastSuccessfulRow == null) {
         LOG.warn("We are restarting the first next() invocation," +
             " if your mapper has restarted a few other times like this" +
             " then you should consider killing this job and investigate" +
             " why it's taking so long.");
       }
       if (lastSuccessfulRow == null) {
         restart(scan.getStartRow());
       } else {
         restart(lastSuccessfulRow);
         scanner.next();    // skip presumed already mapped row
       }
       value = scanner.next();
       if (value != null && value.isStale()) numStale++;
       numRestarts++;
     }
     if (value != null && value.size() > 0) {
       key.set(value.getRow());
       lastSuccessfulRow = key.get();
       return true;
     }

     updateCounters();
     return false;
   } catch (IOException ioe) {
     if (logScannerActivity) {
       long now = System.currentTimeMillis();
       LOG.info("Mapper took " + (now-timestamp)
         + "ms to process " + rowcount + " rows");
       LOG.info(ioe);
       String lastRow = lastSuccessfulRow == null ?
         "null" : Bytes.toStringBinary(lastSuccessfulRow);
       LOG.info("lastSuccessfulRow=" + lastRow);
     }
     throw ioe;
   }
 }
~~~

其实感觉通过直接读HFile可能更好一些。
