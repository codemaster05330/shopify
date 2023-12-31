<template>
  <div class="collection">
    <h3 class="collection__heading">{{ collectionTitle }}</h3>
    <div class="collection__content">
      <div class="collection__sidebar">
        <h6 v-if="$route.params.handle.includes('all-womens') || $route.params.handle.includes('womens')">Shop Women</h6>
        <h6 v-else>Shop Men</h6>
        <nuxt-link v-if="$route.params.handle.includes('all-womens') || $route.params.handle.includes('womens')" v-for="(link, index) in menuLinks.menuLinks.womens" :key="index" :to="`/${$store.state.activeCurrency}${link.url}`">
          <h6>{{link.name}}</h6>
        </nuxt-link>
        <nuxt-link v-if="!$route.params.handle.includes('all-womens') && !$route.params.handle.includes('womens')" v-for="(link, index) in menuLinks.menuLinks.mens" :key="index" :to="`/${$store.state.activeCurrency}${link.url}`">
          <h6>{{link.name}}</h6>
        </nuxt-link>
      </div>
      <div class="product-grid">
        <gridProduct
          v-for="(product, index) in products"
          :key="index"
          :productData="product.node"
        />
      </div>
    </div>
  </div>
</template>

<script>
import Vue from 'vue'
import VueApollo from 'vue-apollo'
import gql from 'graphql-tag'
import GridProduct from '~/components/GridProduct.vue'
import menuLinks from '~/assets/json/menuLinks.json'
import collectionQuery from '~/API/collectionQuery'
import getIllustrations from '~/plugins/getIllustrations'

export default Vue.extend({
  components: {
    GridProduct
  },
  data: () => {
    return {
      menuLinks
    }
  },
  async asyncData (ctx) {
    try {
      if (ctx.app && ctx.app.apolloProvider) {
        const client = ctx.app.apolloProvider.clients[ctx.params.locale]
        let result
        let title = ''
        let collHandle =''
        if (ctx.params.style) {
          const gender = ctx.params.handle.match(/[wo]*mens/g)
          const style = ctx.params.style.replace(/style-/, '')
          title = style.charAt(0).toUpperCase() + style.slice(1)
          const styleQuery = `${gender}, ${style}`
          result = await client.query({
            // Apollo GraphQL query: fetch data
            query: gql`
              query {
                products(first: 120, query: "${styleQuery}") {
                  ${collectionQuery}
                }
              }
            `
          })
        } else {
          result = await client.query({
            // Apollo GraphQL query: fetch data
            query: gql`
              query {
                collectionByHandle(handle: "${ctx.params.handle}") {
                  title,
                  handle
                  products(first: 120) {
                    ${collectionQuery}
                  }
                }
              }
            `
          });
          ({ title, collHandle } = result.data.collectionByHandle.title)
        }

        // New array that only contains the Color product options
        const productsData = result.data.collectionByHandle ? result.data.collectionByHandle.products.edges : result.data.products.edges
        const products = []
        for (let i = 0; i < productsData.length; i++) {
          const a = productsData[i]
          if (a.node.options) {
            // Find the Color product option
            const colorOpt = a.node.options.find(b => b.name === 'Color')
            if (colorOpt) {
              a.node.colorValues = colorOpt.values
            } else {
              a.node.colorValues = []
            }
          } else {
            a.node.colorValues = []
          }

          let bundleTag = ''
          let isSingleProduct = a.node.tags.some((tag) => {
            const isBundleTag = tag.includes('combo') || tag.includes('quant')
            if (isBundleTag) {
              if (tag.split('-').length > 2) {
                return true
              }
              else {
                bundleTag = tag
              }
              return false
            }
          })
          if (!bundleTag) {
            isSingleProduct = true
          }
          // productIllustration for single and quantity products
          // bundleIllustrations for Bundles
          a.node.isSingleProduct = isSingleProduct
          if (isSingleProduct) {
            try {
              // Handle unisex products not having separate illustrations
              let illuHandle = a.node.handle
              if (
                illuHandle.includes('accessories') ||
                illuHandle.includes('socks') ||
                illuHandle.includes('over-shirt')
              ) {
                illuHandle = illuHandle
                  .replace(/womens-/g, '')
                  .replace(/mens-/g, '')
              }

              const productSvg = await import(
                '~/assets/svg/products/' + illuHandle + '.svg?raw'
              )
              if (productSvg.default)
                a.node.productIllustration = productSvg.default
            } catch (err) {
              console.log(err)
              a.node.productIllustration = ''
            }
          } else {
            // Loading illustrations for Bundles
            try {
              // Handle unisex products not having separate illustrations
              if (a.node.description.split('|').length > 2) {
                if (a.node.description.split('|')[1] === 'complete') {
                  // In case of combo bundles with three or more products
                  a.node.completeBundle = true
                  a.node.quantity = null
                } else if (a.node.description.split('|')[1] === 'null') {
                  a.node.quantity = null
                } else {
                  a.node.quantity = a.node.description.split('|')[1]
                }
              } else {
                // In case of combo bundles
                a.node.quantity = null
              }

              let illuHandles = []
              if (a.node.completeBundle) {
                illuHandles = a.node.description
                  .split('|')[0]
                  .split('---')
                  .slice(0, 3)
              } else {
                illuHandles = a.node.description
                  .split('|')[0]
                  .split('---')
                  .slice(0, 2)
              }

              illuHandles.forEach((handle) => {
                if (
                  handle.includes('accessories') ||
                  handle.includes('socks') ||
                  handle.includes('over-shirt')
                ) {
                  handle = handle.replace(/womens-/g, '').replace(/mens-/g, '')
                }
              })

              // This is where the lazy load of the illustrations is triggered
              await getIllustrations(illuHandles).then((data) => {
                // The illustrations for the bundle are added to a new property inside of the bundle
                a.node.bundleIllustrations = data.map(illu => illu.default)
                // We select the first illustration as the main bundle illustration in case of quantity bundles
                a.node.productIllustration = a.node.bundleIllustrations[0]
              })
            } catch (err) {
              console.log(err)
            }
          }
          if (a.node.quantity) {
            for (let i = 0; i < a.node.quantity - 1; i++) {
              if(a.node && a.node.bundleIllustrations) {
                a.node.bundleIllustrations.push(a.node.productIllustration)
              }
            }
          }
          products.push(a)
        }
        // Sort bundles in the bottom
        products.sort((a) => {
          const result = a.node.bundleIllustrations ? 1 : -1
          return result
        })
        // Sort by amount of variants (more or less equals most popular products)
        products.sort((a, b) => {
          if (a.node.variants.edges.length > b.node.variants.edges.length) {
            return -1
          } else {
            return 1
          }
        })

        return {
          // nuxt el : query var
          products,
          collectionTitle: title,
          collHandle
        }
      } else {
        return { products: [], collectionTitle: '', collHandle: '' }
      }
    } catch (err) {
      console.error(err)
      return { products: [], collectionTitle: '', collHandle: '' }
    }
  }
})
</script>

<style lang="scss">
@import "~assets/scss/mixins.scss";

.collection {
  padding-left: 30px;
  padding-right: 30px;
  width: 100%;

  @include screenSizes(tabletPortrait) {
    padding: 0;
  }

  .collection__heading {
    margin: 20px auto;
    text-align: center;

    @include screenSizes(tabletPortrait) {
      text-align: center;
      margin-left: auto !important;
    }

    @include screenSizes(desktopSmall) {
      margin-left: 15%;
    }
  }

  .collection__content {
    display: flex;
    flex-direction: row;
    align-items: flex-start;

    @include screenSizes(phone) {
      width: 100vw;
    }

    .collection__sidebar {
      display: flex;
      flex-direction: column;
      padding: 0 20px;
      position: sticky;
      top: 160.5px;
      width: 15%;

      > h6 {
        font-weight: bold;
        padding-bottom: 5px;
      }

      a {
        padding-bottom: 5px;
      }

      @include screenSizes(tabletPortrait) {
        display: none;
      }
    }
  }

  .product-grid {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    flex-basis: 70%;
    margin: 0 0 20% 0;

    @include screenSizes(desktopSmall) {
      flex-basis: 85%;
    }

    @include screenSizes(tabletPortrait) {
      justify-content: space-between;
      max-width: 100vw;
      padding-left: 25px;
      padding-right: 25px;
      flex-basis: 100%;
      margin: 0;
    }

    @include screenSizes(phone) {
      padding-left: 10px;
      padding-right: 10px;
    }
  }
}
</style>
