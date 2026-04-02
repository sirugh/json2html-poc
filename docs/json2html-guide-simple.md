# Overview

This guide walks you through the process of configuring the json2html worker to render product pages.

## Prerequisites

 - Site onboarded to EDS
 - Helix API access to update site configuration
 - No metadata/bulk metadata for the paths to be served by the json2html worker, at least during set up.

# 1. Commit the template to your repo

<See templates/product-detail.html>

# 2. POST the configuration to the json2html worker

curl --request POST \
	--url https://json2html.adobeaem.workers.dev/config/sirugh/json2html-poc/main \
	--header 'Authorization: token <TOKEN_HERE>' \
	--header 'Content-Type: application/json' \
	--data '[
		{
		"path": "/products/",
		"endpoint": "https://www.aemshop.net/cs-graphql?query=query+GET_PRODUCT_DATA%28%24skus%3A+%5BString%5D%29+%7B+products%28skus%3A+%24skus%29+%7B+...PRODUCT_FRAGMENT+%7D+%7D+fragment+PRODUCT_FRAGMENT+on+ProductView+%7B+__typename+id+sku+name+shortDescription+metaDescription+metaKeyword+metaTitle+description+inStock+addToCartAllowed+url+urlKey+externalId+images%28roles%3A+%5B%5D%29+%7B+url+label+roles+%7D+videos+%7B+description+url+title+preview+%7B+label+roles+url+%7D+%7D+attributes%28roles%3A+%5B%5D%29+%7B+name+label+value+roles+%7D+...+on+SimpleProductView+%7B+price+%7B+roles+regular+%7B+amount+%7B+value+currency+%7D+%7D+final+%7B+amount+%7B+value+currency+%7D+%7D+tiers+%7B+tier+%7B+amount+%7B+value+currency+%7D+%7D+quantity+%7B+...+on+ProductViewTierRangeCondition+%7B+gte+lt+%7D+%7D+%7D+%7D+%7D+...+on+ComplexProductView+%7B+options+%7B+...PRODUCT_OPTION_FRAGMENT+%7D+...PRICE_RANGE_FRAGMENT+%7D+%7D+fragment+PRODUCT_OPTION_FRAGMENT+on+ProductViewOption+%7B+id+title+required+multi+values+%7B+id+title+inStock+__typename+...+on+ProductViewOptionValueProduct+%7B+title+quantity+isDefault+__typename+product+%7B+sku+shortDescription+metaDescription+metaKeyword+metaTitle+name+price+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+%7D+%7D+...+on+ProductViewOptionValueSwatch+%7B+id+title+type+value+inStock+%7D+%7D+%7D+fragment+PRICE_RANGE_FRAGMENT+on+ComplexProductView+%7B+priceRange+%7B+maximum+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+minimum+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+%7D+%7D&variables=%7B%22skus%22%3A%5B%22{{headers['"'"'x-content-source-location'"'"']}}%22%5D%7D",
		"template": "/templates/product-detail.html",
		"headers": {
		"Magento-Store-Code": "main_website_store",
		"Magento-Store-View-Code": "default",
		"Magento-Website-Code": "base",
		"x-api-key": "4dfa19c9fe6f4cccade55cc5b3da94f7",
		"Magento-Environment-Id": "f38a0de0-764b-41fa-bd2c-5bc2f3c7b39a",
		"Content-Type": "application/json"
		},
		"forwardHeaders": ["x-content-source-location"]
		}
	]'

# 3. Check the json2html worker response, to ensure it is configured correctly

curl 'https://json2html.adobeaem.workers.dev/sirugh/json2html-poc/main/products/bezier-mega-tumbler/adb247' \
	-H 'x-content-source-location: ADB247'


# 4. Preview the Page

curl --request POST \
  --url https://admin.hlx.page/preview/sirugh/json2html-poc/main/products/bezier-mega-tumbler/ADB247 \
  --header 'x-auth-token: <TOKEN_HERE>' \
  --header 'x-content-source-location: ADB247'

# 5. Inspect and validate the preview page

Open https://main--json2html-poc--sirugh.aem.page/products/bezier-mega-tumbler/adb247 and inspect the initial HTML response from the EDS origin.
It should contain content from the json2html worker, such as product name, etc.
