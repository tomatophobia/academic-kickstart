---
# 사용하지 않으므로 빌드 막아놓음
cms_exclude: true

_build:
  render: never
cascade:
  _build:
    render: never
    list: always

# Generate Wowchemy CMS
type: wowchemycms
outputs:
- wowchemycms_config
- HTML
---
