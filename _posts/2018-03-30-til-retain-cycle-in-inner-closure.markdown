---
layout: post
title:  "#til Retain cycle in inner closure"
date:   2018-03-30 10:03:00 +0800
categories: til programming
tags: [swift, programming, rxdatasources]
---
Take a look at below code, my *FeedViewController*, I use [RxDataSource][rxdatasources-gh] to bind data to *UITableView*, Inside the cell, I have button and use a closure callback to handle tap event.


```swift
let dataSource = RxTableViewSectionedReloadDataSource<SectionOfFeedData>(configureCell: { (ds, tableView, indexPath, post) -> UITableViewCell in
    let cell: FeedTableViewCell
    switch post.postType {
	case .buy:
		cell = tableView.dequeueReusableCell(withIdentifier: "FeedCellBuy", for: indexPath) as! FeedTableViewCell
	case .rent:
		cell = tableView.dequeueReusableCell(withIdentifier: "FeedCellRent", for: indexPath) as! FeedTableViewCell
    }
    cell.configure(with: post)

    // Event handling, I need to access `self` and don't want retain cycle. Let's use [weak self]!
    cell.onTap = { [weak self] post in
        self?.goToContactAgent(post: post)
    }

    return cell
})

viewModel.filteredPosts
    .asDriver()
    .map { [SectionOfFeedData(items: $0)] }
    .drive(tableView.rx.items(dataSource: dataSource))
    .disposed(by: disposeBag)
```

Then I found out my *FeedViewController* got retain cycle, and after hours debug, comment out codes,... I found root cause is from the above code. 

It looks like the **[weak self]** in *onTap* block uses the **self** from its context - the *configureCell* block, which mean it's a **strong self**. So put **[weak self]** in **configureCell** block instead. And it worked, my *ViewController* can deinit!!! That's all I need.

![Problem Sovled]({{ "/assets/problem-solved.jpg" | absolute_url }})


**Be more aware to your code, especially memory management.**

[rxdatasources-gh]: https://github.com/RxSwiftCommunity/RxDataSources