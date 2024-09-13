---
title: "About me"
date: 2022-03-06
# layout: "archives"
slug: "about-me "
resources:
- name: pdf-file-:counter
  src: '**.pdf'
menu:
    main:
        weight: 6
        params: 
            icon: user
---

<style>
    .pdf-container {
        width: 100%;
        max-width: 800px;
        margin: 20px auto;
        padding: 10px;
        background-color: #fff;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
        border-radius: 8px;
        overflow: hidden; /* Hide overflow to prevent scrollbars */
    }
 
   iframe{
    overflow:hidden;
}
</style>
 [Download](./cv.pdf)


<div class="pdf-container">

<iframe   frameborder="0"   scrolling="auto" seamless="seamless"  src="./cv.pdf#toolbar=0&navpanes=0&scrollbar=0" style="height:1800px;width:100%;border:none;" type="application/pdf" ></iframe>
</div>
