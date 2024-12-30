---
title: "CV"
meta_title: "CV"
description: "this is meta description"
image: "/images/avatar.jpg"
draft: false
---


<div id="adobe-dc-view" style="height: 100vh; width: 80vh;"></div>
<script src="https://acrobatservices.adobe.com/view-sdk/viewer.js"></script>
<script type="text/javascript">
  document.addEventListener("adobe_dc_view_sdk.ready", function(){
    var adobeDCView = new AdobeDC.View({clientId: "e888a96aa42146bea180538e306d7e31", divId: "adobe-dc-view"}); // this client id only work for the "nicholas-bn.github.io" domain
    adobeDCView.previewFile({
      content:{ location:
        { url: "./CV_Nicho_EN_dec_2024_without_mail_tel.pdf"}},
      metaData:{fileName: "Nicholas's CV"}
    },
    {
      embedMode: "SIZED_CONTAINER"
    });
  });
</script>
