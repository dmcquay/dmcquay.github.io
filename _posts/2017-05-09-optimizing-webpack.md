---
layout: post
title: "Optimizing Webpack"
date: 2017-05-09
categories: Optimization
---

NODE_ENV=production webpack --json > stats.json

https://chrisbateman.github.io/webpack-visualizer/

momentjs -> locale is huge and often unused
if you don't need locale, instruct webpack to exclude it:
plugins: [
	...
	new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/)
]

lodash/ramda -> use babel plugins or import only the module you need

babel-plugin-lodash

or just import modules individually

import _ from 'lodash'
import { add } from 'lodash/fp/add'
import { map } from 'lodash/map

lodash additionally has webpack plugin that optimizes a bit more: lodash-webpack-plugin

ramda has a babel plugin too: babel-plugin-ramda

or import directly:
import add from 'ramda/src/add'
