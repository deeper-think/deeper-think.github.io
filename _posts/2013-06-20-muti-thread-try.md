---
layout: post
title: linux 多线程编程初试
category: 编程调试技巧
tags: [多线程]
keywords: 多线程
description: 
---

> 用正确的工具，做正确的事情

基于多线程实现的经典的“生产值-消费者”模型，假设生产缓冲区为1，详细实现如下：

	#include <stdlib.h>
	#include <stdio.h>
	#include <pthread.h>
	
	int product = 0;
	pthread_cond_t buf_ready = PTHREAD_COND_INITIALIZER;
	pthread_cond_t prod_ready = PTHREAD_COND_INITIALIZER;
	pthread_mutex_t prod_lock = PTHREAD_MUTEX_INITIALIZER;
	
	void *producer(void *apue) {
	    printf("start producer!\n");
	    while(1) {
	        pthread_mutex_lock(&prod_lock);
	        if(1 == product)
	            pthread_cond_wait(&buf_ready, &prod_lock);
	        product = 1;
	        sleep(1);
	        printf("put a product!\n");
	        pthread_cond_signal(&prod_ready);
	        pthread_mutex_unlock(&prod_lock);
	    }
	}
	
	void *consumer(void *apue) {
	    printf("start consumer!\n");
	    while(1) {
	        pthread_mutex_lock(&prod_lock);
	        if(0 == product)
	            pthread_cond_wait(&prod_ready, &prod_lock);
	        product = 0;
	        sleep(1);
	        printf("get a product!\n");
	        pthread_cond_signal(&buf_ready);
	        pthread_mutex_unlock(&prod_lock);
	    }
	}
	
	int 
	main(void) {
	    pthread_t producer_tid, consumer_tid;
	    void *tret;
	    int err;
	    
	    pthread_create(&producer_tid, NULL, producer, NULL);
	    pthread_create(&consumer_tid, NULL, consumer, NULL);
	    pthread_join(producer_tid, &tret);
	    pthread_join(consumer_tid, &tret);
	    return 0;
	}

玩的开心 !!!
