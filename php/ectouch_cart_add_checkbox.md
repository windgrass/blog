# 购物车勾选结算功能
开发使用的是ectouch企业版，带有购物车功能。
当前功能：在商品详情页面，点击添加购物车后，会进入购物车中。在购物车页面，点击‘立即购买’会将购物车中的全部商品都结算生成订单。  
项目需求：添加购物中的商品前面带有‘勾选框’，点击已经勾选的商品才结算，其他的继续待在购物车中。

-----------------------------
## 代码分析
查看ectouch代码，分析：
1. 商品如何进入购物车表
2. 购物车数量修改
3. 订单生成


### 商品进入购物车
前端页面点击改变商品数量，ajax方式发送商品id，数量到后台控制器  
后台根据接收到的json数据添加向购物车中添加商品

### 购物车数量修改
每次点击‘+-’都会给后台发送一次ajax请求，实时更新数据表中的商品数量


### 订单生成
查看代码发现，在结算生成订单的时候，是依据当前用户Id，去查询购物车表中的商品，然后查询商品价格信息，全部进行结算的。

-----------------------------
## 改造思路
分析后决定添加一个表，叫做‘虚拟车’，原购物车叫做‘结算车’，专门放购物车商品信息，不参与结算。勾选的商品再添加入购物车表，结算的时候会根据购物车中的商品结算。  

具体步骤：
1. 新建‘虚拟车’表
2. 商品详情页面的‘加入购物车’按钮，改成添加到虚拟车
3. 购物车商品列表，显示虚拟车的数据
4. 购物车增加勾选框，商品数据修改都是修改虚拟车
5. 点击‘立即购买’，前端根据已勾选的商品数据发送ajax请求，添加到结算车
6. 订单确认页面是根据结算车中的商品数据生成的，提交订单后需删除虚拟车中对应的商品
7. 提交订单，会直接访问done()，在订单生成后，会有个清空购物车的操作，在这个操作之前，添加一个清空虚拟车的操作。

-------------------------------
## 具体代码示例
这里贴出从进入购物车到结算的整个代码的大概流程  

### 商品详情页添加购物车
商品详情页，点击加入购物车会有弹出框，选择数量，点击‘确定’添加到购物车  
确定按钮的代码：
```
<a type="button" class="btn btn-block btn-primary text-center" style="background: #20A3DD;" href="javascript:addToCartNew_quick({$goods.goods_id}, 0, 0, '{$rfansid}')">确定</a>
```
这里调用了addToCartNew_quick，在common.js文件中，代码中注释的为修改为虚拟车之前的代码  
代码如下：
```
function addToCartNew_quick(goodsId, parentId, sid, rfansid) {
    var goods = new Object();
    var spec_arr = new Array();
    var fittings_arr = new Array();
    var number = 1;
    var formBuy = document.forms['ECS_FORMBUY'];
    var quick = 0;

    // 检查是否有商品规格
    if (formBuy) {
        str = getSelectedAttributes(formBuy);
        spec_arr = str.split(',');
        if (formBuy.elements['number']) {
            number = formBuy.elements['number'].value;
        }
        quick = 1;
    }

    goods.quick = quick;
    goods.spec = spec_arr;
    goods.goods_id = goodsId;
    goods.number = number;
    goods.parent = (typeof (parentId) == "undefined") ? 0 : parseInt(parentId);
    goods.sceneid = sid;
    goods.rfansid = rfansid;
    $.post('index.php?m=default&c=flow&a=add_virtual_cart', {
        goods: $.toJSON(goods)
    }, function(data) {
        addToCartResponse_quick(data);
    }, 'json');
    //Ajax.call('index.php?m=default&c=flow&a=add_to_cart', 'goods=' + goods.toJSONString(), addToCartResponse_quick, 'POST', 'JSON');
}
```
后台的add_virtual_cart方法是新添加的，内部逻辑是仿照原有的add_to_cart，包括数据表的结构，也是参照原有的购物车表  
代码如下：
```
public function add_virtual_cart()
    {
        //对goods处理
        $_POST ['goods'] = strip_tags(urldecode($_POST ['goods']));
        $_POST ['goods'] = json_str_iconv($_POST ['goods']);
        if (!empty($_REQUEST ['goods_id']) && empty($_POST ['goods'])) {
            if (!is_numeric($_REQUEST ['goods_id']) || intval($_REQUEST ['goods_id']) <= 0) {
                ecs_header("Location:./\n");
            }
            $goods_id = intval($_REQUEST ['goods_id']);
            exit();
        }
        // 初始化返回数组
        $result = array(
            'error' => 0,
            'message' => '',
            'content' => '',
            'goods_id' => '',
            'product_spec' => ''
        );

        if (empty($_POST ['goods'])) {      //没有goods参数
            $result ['error'] = 1;
            die(json_encode($result));
        }
        $json = new EcsJson;
        $goods = $json->decode($_POST ['goods']);   //解析商品json数据
        $result['goods_id'] = $goods->goods_id;
        $result['product_spec'] = $goods->spec;


        // 查询：系统启用了库存，检查输入的商品数量是否有效
        // 查询
        $arrGoods = $this->model->table('goods')->field('goods_name,goods_number,extension_code')->where('goods_id =' . $goods->goods_id)->find();
        $goodsnmber = model('Users')->get_goods_number($goods->goods_id);
        $goodsnmber+=$goods->number;
        if (intval(C('use_storage')) > 0 && $arrGoods ['extension_code'] != 'package_buy') {
            if ($arrGoods ['goods_number'] < $goodsnmber) {
                $result['error'] = 1;
                $result['message'] = sprintf(L('stock_insufficiency'), $arrGoods ['goods_name'], $arrGoods ['goods_number'], $arrGoods ['goods_number']);
                if (C('use_how_oos') == 1){
                    $result['message'] =L('oos_tips');
                }
                die(json_encode($result));
            }
        }

        // 检查：商品数量是否合法
        if (!is_numeric($goods->number) || intval($goods->number) <= 0) {
            $result ['error'] = 1;
            $result ['message'] = L('invalid_number');
        } else {
            // 更新：添加商品到到 虚拟车
            if (model('Order')->addto_virtual_cart($goods->goods_id, $goods->number, $goods->spec, $goods->parent, $goods->sceneid, $goods->rfansid)) {     // TODO 在前端发送的json数据中添加$goods->rfansid
                if (C('cart_confirm') > 2) {
                    $result ['message'] = '';
                } else {
                    $result ['message'] = C('cart_confirm') == 1 ? L('addto_cart_success_1') : L('addto_cart_success_2');
                }
                $result ['content'] = insert_cart_info();       // 此处是当前已勾选商品的信息
                $result ['one_step_buy'] = C('one_step_buy');
            } else {
                $result ['message'] = ECTouch::err()->last_message();
                $result ['error'] = ECTouch::err()->error_no;
                $result ['goods_id'] = stripslashes($goods->goods_id);
                if (is_array($goods->spec)) {
                    $result ['product_spec'] = implode(',', $goods->spec);
                } else {
                    $result ['product_spec'] = $goods->spec;
                }
            }
        }
        $cart_confirm = C('cart_confirm');
        $result ['confirm_type'] = !empty($cart_confirm) ? C('cart_confirm') : 2;
        // 返回购物车商品总数量
        $result ['cart_number'] = insert_cart_info_number();        // 此处是当前已勾选商品的信息
        die(json_encode($result));
    }
```
后台返回的是一个json数据，addToCartNew_quick中发送ajax后，调用addToCartResponse_quick来处理返回值  
代码如下：
```
/* *
 * 处理添加商品到购物车的反馈信息
 */
function addToCartResponse_quick(result) {
    if (result.error > 0) {
        // 如果需要缺货登记，跳转
        if (result.error == 1) {
			if(use_how_oos == 1){
        		if (confirm(result.message)) {
    				location.href = 'index.php?m=default&c=user&a=add_booking&id='
    						+ result.goods_id + '&spec=' + result.product_spec;
    			}
        	}else{
        		alert(result.message);
        	}
			
        }
        // 没选规格，弹出属性选择框
        else if (result.error == 6) {
            openSpeDiv(result.message, result.goods_id, result.parent);
        } else {
            alert(result.message);
        }
    } else {
        var cartInfo = document.getElementById('ECS_CARTINFO');
        var cart_url = 'index.php?m=default&c=flow&a=cart';
        if (cartInfo) {
            cartInfo.innerHTML = result.content;
        }
        $('#model_sure').fadeIn(100);
        $('.to_cart').on('click',function () {
            $('#model_sure').fadeOut(100);
            location.href = cart_url;
        })
        // var returnVal = window.confirm("商品已成功加入购物车\n是否去购物车查看？", "标题");
        // if (returnVal) {
        //     location.href = cart_url;
        // }

    }
```
如果没有报错，会直接跳转到‘index.php?m=default&c=flow&a=cart’，也就是购物车页面
```
    /**
     * 购物车列表
     */
    public function index() {
		$_SESSION['flow_type'] = CART_GENERAL_GOODS;
        /* 如果是一步购物，跳到结算中心 */
        if (C('one_step_buy') == '1') {
            ecs_header("Location: " . url('flow/checkout') . "\n");
        }

        // 取得商品列表，计算合计
//        $cart_goods = model('Order')->get_cart_goods();                   //修改购物车逻辑-----20170315--Dennis
        $cart_goods = model('Order')->get_cart_virtual_goods();             //显示的是 虚拟车 中的数据
        $this->assign('goods_list', $cart_goods ['goods_list']);
        $this->assign('total', $cart_goods ['total']);

        if ($cart_goods['goods_list']) {
            // 相关产品
            $linked_goods = model('Goods')->get_linked_goods($cart_goods ['goods_list']);
            $this->assign('linked_goods', $linked_goods);
        }

        // 购物车的描述的格式化
        $this->assign('shopping_money', sprintf(L('shopping_money'), $cart_goods ['total'] ['goods_price']));
        $this->assign('market_price_desc', sprintf(L('than_market_price'), $cart_goods ['total'] ['market_price'], $cart_goods ['total'] ['saving'], $cart_goods ['total'] ['save_rate']));

        // 取得优惠活动
        $favourable_list = model('Flow')->favourable_list_flow($_SESSION ['user_rank']);
        usort($favourable_list, array("FlowModel", "cmp_favourable"));
        $this->assign('favourable_list', $favourable_list);

        // 计算折扣
        $discount = model('Order')->compute_discount();
        $this->assign('discount', $discount ['discount']);

        // 折扣信息
        $favour_name = empty($discount ['name']) ? '' : join(',', $discount ['name']);
        $this->assign('your_discount', sprintf(L('your_discount'), $favour_name, price_format($discount ['discount'])));

        // 增加是否在购物车里显示商品图
        $this->assign('show_goods_thumb', C('show_goods_in_cart'));

        // 增加是否在购物车里显示商品属性
        $this->assign('show_goods_attribute', C('show_attr_in_cart'));


        // 取得购物车中基本件ID
//        $condition = "session_id = '" . SESS_ID . "' " . "AND rec_type = '" . CART_GENERAL_GOODS . "' " . "AND is_gift = 0 " . "AND extension_code <> 'package_buy' " . "AND parent_id = 0 ";
//        $parent_list = $this->model->table('cart')->field('goods_id')->where($condition)->getCol();
//
//      //修改购物车逻辑-----20170315--Dennis
        $condition = "user_id = '" . $_SESSION['user_id'] . "' " . "AND rec_type = '" . CART_GENERAL_GOODS . "' " . "AND is_gift = 0 " . "AND extension_code <> 'package_buy' " . "AND parent_id = 0 ";
        $parent_list = $this->model->table('cart_virtual')->field('goods_id')->where($condition)->getCol();
        //根据基本件id获取 购物车中商品配件列表
        $fittings_list = model('Goods')->get_goods_fittings($parent_list);
        $this->assign('fittings_list', $fittings_list);
        $this->assign('currency_format', C('currency_format'));
        $this->assign('integral_scale', C('integral_scale'));
        $this->assign('step', 'cart');
        $this->assign('title', L('shopping_cart'));
        $this->display('flow.dwt');
    }
```
购物车页面添加勾选框

```
<ul class="goods_list_box">
    <!-- {foreach from=$goods_list item=goods key=k} -->
    <li style="position: relative;" class="active">
        <input type="hidden" name="sceneid" value="{$goods.sceneid}" class="goods_sceneid"/>
        <input type="hidden" name="refer_fansid" value="{$goods.refer_fansid}" class="refer_fansid"/>
        <div class="ect-clear-over">
            <i class="click pull-left"></i>     <!-- 这里添加勾选框 -->
            <a href="{:url('goods/index',array('id'=>$this->_var['goods']['goods_id']))}">
                <img class="img-responsive" src="{$goods.goods_thumb}" title="{$goods.goods_name|escape:html}">
            </a>
            <dl>
                <dt>
                <h4 class="title" style="font-size: 1em;"><a href="{:url('goods/index',array('id'=>$this->_var['goods']['goods_id']))}" style="width: 100%;">{$goods.goods_name}</a></h4>

                </dt>
                <!-- {if $goods.parent_id gt 0} 配件 -->
                <span style="color:#ffae00">{$lang.accessories}</span>
                <!-- {/if} -->
                <!-- {if $goods.is_gift gt 0} 赠品 -->
                <span style="color:#ffae00">{$lang.largess}</span>
                <!-- {/if} -->
                <dd class="ect-color999">
                    <!-- {if $show_goods_attribute eq 1} 显示商品属性 -->
                    <p>{$goods.goods_attr|nl2br}</p>
                    <!-- {/if} -->
                    <div class="pull-right flow-del text-center" style="position: absolute; bottom: 0.6em; right: 0px;">
                        <a class="" href="javascript:if (confirm('{$lang.drop_goods_confirm}')) location.href='{:url('flow/drop_virtual_cart_goods',array('id'=>$this->_var['goods']['rec_id']))}';" >
                            <img class="delete" src="__TPL__/images/delete.png" alt="">
                        </a>
                    </div>
                    <div class="ect-margin-tb ect-margin-bottom0 ect-clear-over flow-num-del" style="position: absolute; bottom: 0.6em; ">
                        <p><strong class="ect-colory">{$goods.goods_price}</strong></p>

                        <!-- {if $goods.goods_id gt 0 && $goods.is_gift eq 0 && $goods.parent_id eq 0} 普通商品可修改数量 -->
                        <div class="input-group pull-left wrap"> <span class="input-group-addon" onClick="change_goods_number('1',{$goods.rec_id})" >-</span>
                            <input type="hidden" id="back_number{$goods.rec_id}" value="{$goods.goods_number}" />
                            <input type="text" class="form-num form-contro"  name="{$goods.rec_id}" id="goods_number{$goods.rec_id}" autocomplete="off" value="{$goods.goods_number}" onFocus="back_goods_number({$goods.rec_id})"  onblur="change_goods_number('2',{$goods.rec_id})" />
                            <span class="input-group-addon" onClick="change_goods_number('3',{$goods.rec_id})">+</span> </div>

                        <!-- {else} -->
                        数量{$goods.goods_number}
                        <!-- {/if} -->
                    </div>
                </dd>
            </dl>
        </div>
    </li>
    <!-- {/foreach} -->
</ul>
</section>
<div class="col-xs-12">
    <a type="botton" class="btn-block btn-lg btn-default text-center" style="color: #fff;">立即购买 <span class="btn-block_box">您还没有选择商品哦！</span></a>
</div>
```
点击切换图标的效果，是在<i class="click pull-left"></i>上绑定了点击事件
```
    //选中/不选中商品
    function onchoice() {
        $('.click').on('click',function () {
            var a=$(this).parent().parent();
            if(a.hasClass('active')){
                a.removeClass('active');
            }else {
                a.addClass('active');
            }
            offSure();
        })
    }
    onchoice();
```
对商品数量的修改也更改了，点击‘+-’会调用change_goods_number
```
//实时更新数据
    function change_goods_number(type, id) {
        var goods_number =document.getElementById('goods_number' + id).value;
//        console.log(goods_number);
            if (type != 2) {
                back_goods_number(id)
            }
            if (type == 1) {
                goods_number--;
            }
            if (type == 3) {
                goods_number++;
            }
            if (goods_number <= 0) {
                goods_number = 1;
            }
            //防止输入非法数量----测试结结果是否为数字
            if (!/^[0-9]*$/.test(goods_number)) {
                goods_number = document.getElementById('goods_number' + id).value;
            }
        if (goods_number===undefined){
            //发请求修改数据库的商品数量
            document.getElementById('goods_number' + id).value = 0;
            $.post('{:url("flow/ajax_update_virtual_cart")}', {
                rec_id: id, goods_number: 0
            }, function (data) {
                change_goods_number_response(data, id);
            }, 'json');
        }else {
            //var goods_number = document.getElementById('goods_number' + id).value;
            document.getElementById('goods_number' + id).value = goods_number;
            $.post('{:url("flow/ajax_update_virtual_cart")}', {
                rec_id: id, goods_number: goods_number
            }, function (data) {
                change_goods_number_response(data, id);
            }, 'json');
        }

    }
```
在这里面，还有调用了一个函数，获取当前的商品数量
```
    //获取需要提交的信息
    function back_goods_number(id) {
        var goods_number = document.getElementById('goods_number' + id).value;
//        console.log(goods_number);
        if(goods_number!==undefined){
            document.getElementById('back_number' + id).value = goods_number;
        }else {
            document.getElementById('back_number' + id).value = 1;
        }
    }
```
更改购物车数据，访问的是后台接口{:url("flow/ajax_update_virtual_cart")}
```
public function ajax_update_virtual_cart() {
        //格式化返回数组
        $result = array(
            'error' => 0,
            'message' => ''
        );
        // 是否有接收值
        if (isset($_POST ['rec_id']) && isset($_POST ['goods_number'])) {
            $key = $_POST ['rec_id'];
            $val = $_POST ['goods_number'];
            $val = intval(make_semiangle($val));
            if ($val <= 0 && !is_numeric($key)) {
                $result ['error'] = 99;
                $result ['message'] = '';
                die(json_encode($result));
            }
            // 查询：
            $condition = " rec_id='$key' AND user_id='" . $_SESSION['user_id'] . "'";
            $goods = $this->model->table('cart_virtual')->field('goods_id,goods_attr_id,product_id,extension_code')->where($condition)->find();

            $sql = "SELECT g.goods_name,g.goods_number " . "FROM " . $this->model->pre . "goods AS g, " . $this->model->pre . "cart_virtual AS c " . "WHERE g.goods_id =c.goods_id AND c.rec_id = '$key'";
            $res = $this->model->query($sql);
            $row = $res[0];
            // 查询：系统启用了库存，检查输入的商品数量是否有效
            if (intval(C('use_storage')) > 0 && $goods ['extension_code'] != 'package_buy') {
                if ($row ['goods_number'] < $val) {
                    $result ['error'] = 1;
                    $result ['message'] = sprintf(L('stock_insufficiency'), $row ['goods_name'], $row ['goods_number'], $row ['goods_number']);
                    $result ['err_max_number'] = $row ['goods_number'];
                    die(json_encode($result));
                }
                /* 是货品 */
                $goods ['product_id'] = trim($goods ['product_id']);
                if (!empty($goods ['product_id'])) {
                    $condition = " goods_id = '" . $goods ['goods_id'] . "' AND product_id = '" . $goods ['product_id'] . "'";
                    $product_number = $this->model->table('products')->field('product_number')->where($condition)->getOne();
                    if ($product_number < $val) {
                        $result ['error'] = 2;
                        $result ['message'] = sprintf(L('stock_insufficiency'), $row ['goods_name'], $product_number, $product_number);
                        die(json_encode($result));
                    }
                }
            } elseif (intval(C('use_storage')) > 0 && $goods ['extension_code'] == 'package_buy') {
                if (model('Order')->judge_package_stock($goods ['goods_id'], $val)) {
                    $result ['error'] = 3;
                    $result ['message'] = L('package_stock_insufficiency');
                    die(json_encode($result));
                }
            }
            /* 查询：检查该项是否为基本件 以及是否存在配件 */
            /* 此处配件是指添加商品时附加的并且是设置了优惠价格的配件 此类配件都有parent_idgoods_number为1 */
            $sql = "SELECT b.goods_number,b.rec_id
			FROM " . $this->model->pre . "cart_virtual a, " . $this->model->pre . "cart_virtual b
				WHERE a.rec_id = '$key'
				AND a.user_id = '" . $_SESSION['user_id'] . "'
			AND a.extension_code <>'package_buy'
			AND b.parent_id = a.goods_id
			AND b.user_id = '" . $_SESSION['user_id'] . "'";

            $offers_accessories_res = $this->model->query($sql);

            // 订货数量大于0
            if ($val > 0) {
                /* 判断是否为超出数量的优惠价格的配件 删除 */
                $row_num = 1;
                foreach ($offers_accessories_res as $offers_accessories_row) {
                    if ($row_num > $val) {
                        $sql = "DELETE FROM" . $this->model->pre . "cart_virtual WHERE user_id = '" . $_SESSION['user_id'] . "' " . " AND rec_id ='" . $offers_accessories_row ['rec_id'] . "' LIMIT 1";
                        $this->model->query($sql);
                    }

                    $row_num++;
                }

                /* 处理超值礼包 */
                if ($goods ['extension_code'] == 'package_buy') {
                    // 更新购物车中的商品数量
                    $sql = "UPDATE " . $this->model->pre . "cart_virtual SET goods_number= '$val' WHERE rec_id='$key' AND user_id='" . $_SESSION['user_id'] . "'";
                } /* 处理普通商品或非优惠的配件 */ else {
                    $attr_id = empty($goods ['goods_attr_id']) ? array() : explode(',', $goods ['goods_attr_id']);
                    $goods_price = model('GoodsBase')->get_final_price($goods ['goods_id'], $val, true, $attr_id);

                    // 更新购物车中的商品数量
                    $sql = "UPDATE " . $this->model->pre . "cart_virtual SET goods_number= '$val', goods_price = '$goods_price' WHERE rec_id='$key' AND user_id='" . $_SESSION['user_id'] . "'";
                }
            }  // 订货数量等于0
            else {
                /* 如果是基本件并且有优惠价格的配件则删除优惠价格的配件 */
                foreach ($offers_accessories_res as $offers_accessories_row) {
                    $sql = "DELETE FROM " . $this->model->pre . "cart_virtual WHERE session_id= '" . SESS_ID . "' " . "AND rec_id ='" . $offers_accessories_row ['rec_id'] . "' LIMIT 1";
                    $this->model->query($sql);
                }

                $sql = "DELETE FROM " . $this->model->pre . "cart_virtual WHERE rec_id='$key' AND user_id='" . $_SESSION['user_id'] . "'";
            }

            $this->model->query($sql);
            /* 删除所有赠品 */
            $sql = "DELETE FROM " . $this->model->pre . "cart_virtual WHERE user_id = '" . $_SESSION['user_id'] . "' AND is_gift <> 0";
            $this->model->query($sql);

            $result ['rec_id'] = $key;
            $result ['goods_number'] = $val;
            $result ['goods_subtotal'] = '';
            $result ['total_desc'] = '';
            $result ['cart_info'] = insert_cart_info();
            /* 计算合计 */
//            $cart_goods = model('Order')->get_cart_goods();
//            foreach ($cart_goods ['goods_list'] as $goods) {
//                if ($goods ['rec_id'] == $key) {
//                    $result ['goods_subtotal'] = $goods ['subtotal'];
//                    break;
//                }
//            }
//            $market_price_desc = sprintf(L('than_market_price'), $cart_goods ['total'] ['market_price'], $cart_goods ['total'] ['saving'], $cart_goods ['total'] ['save_rate']);
            /* 计算折扣 */
//            $discount = model('Order')->compute_discount();
//            $favour_name = empty($discount ['name']) ? '' : join(',', $discount ['name']);
//            $your_discount = sprintf('', $favour_name, price_format($discount ['discount']));
//            $result ['total_desc'] = $cart_goods ['total'] ['goods_price'];
//            $result ['total_number'] = $cart_goods ['total'] ['total_number'];
//            $result['market_total'] =  $cart_goods['total']['market_price'];//市场价格
            die(json_encode($result));
        } else {
            $result ['error'] = 100;
            $result ['message'] = '';
            die(json_encode($result));
        }
    }
```
根据返回信息，修改页面上的商品数量
```
// 处理返回信息并显示
function change_goods_number_response(result, id) {
    if (result.error == 0) {
        var rec_id = result.rec_id;
        $("#goods_number_" + rec_id).val(result.goods_number);
        //document.getElementById('total_number').innerHTML = result.total_number;            //更新数量
        //document.getElementById('goods_subtotal').innerHTML = result.total_desc;            //更新小计
        if (document.getElementById('ECS_CARTINFO')) {
            //更新购物车数量
            document.getElementById('ECS_CARTINFO').innerHTML = result.cart_info;
        }
    } else if (result.message != '') {
        alert(result.message);
        var goods_number =document.getElementById('goods_number' + id).value;
        document.getElementById('goods_number' + id).value = goods_number;
    }
}
```
点击‘立即购买’按钮，触发事件
```
    $('.btn-block').on('click',function () {
        ToCart_quick();
    })
```
ToCart_quick代码，在这里，会获取所有的 class=active，将goods数据添加到data中，然后发送到后台
```
/* *
 * 下单------------2017 03 14 添加-----------解决购物车商品选中才能执行下一步操作
 */
function ToCart_quick(parentId) {
     var data=[];
    $('.active').each(function () {
        var goods = new Object();
        var box= new Object();
        var spec_arr = new Array();
        var fittings_arr = new Array();
        var number = $(this).find('.form-num').val();
        var id=$(this).find('.form-num').attr('id').replace(/[^0-9]/ig,"");
        var goodsId = $(this).find('.ect-clear-over').find('a').attr('href').split('&')[3].replace(/[^0-9]/ig,"");
        // console.log(number);
        // console.log(id);
        // console.log(goodsId);
        box[id]=goodsId;
        var formBuy = document.forms['ECS_FORMBUY'];
        var quick = 0;
        var sceneid = $(this).find('.goods_sceneid').val();
        var rfansid = $(this).find('.refer_fansid').val();

        // 检查是否有商品规格
        if (formBuy) {
            str = getSelectedAttributes(formBuy);
            spec_arr = str.split(',');
            if (formBuy.elements['number']) {
                number = formBuy.elements['number'].value;
            }
            quick = 1;
        }

        goods.quick = quick;
        goods.spec = spec_arr;
        goods.goods_id = goodsId;
        goods.rec_id= id;
        goods.number = number;
        goods.parent = (typeof (parentId) == "undefined") ? 0 : parseInt(parentId);
        goods.sceneid = sceneid;
        goods.rfansid = rfansid;
        data.push(goods);
    })
        $.post('index.php?m=default&c=flow&a=cart_checkout', {
            goods: $.toJSON(data)
        }, function(data) {
            if(data.error==0){
                //console.log(data);
                location.href="/index.php?m=default&c=flow&a=checkout";
            }
        }, 'json');
    //console.log($.toJSON(data));
    //Ajax.call('index.php?m=default&c=flow&a=add_to_cart', 'goods=' + goods.toJSONString(), addToCartResponse_quick, 'POST', 'JSON');
}
```
向后台index.php?m=default&c=flow&a=cart_checkout发送了goods数据后，后台依据发送的数据添加到结算车（添加前先清空当前用户的结算车）
```
public function cart_checkout()
{
    $all_goods = $_POST['goods'];

    $all_goods = strip_tags(urldecode($all_goods));
    $all_goods = json_str_iconv($all_goods);
    if (!empty($_REQUEST ['goods_id']) && empty($all_goods)) {
        if (!is_numeric($_REQUEST ['goods_id']) || intval($_REQUEST ['goods_id']) <= 0) {
            ecs_header("Location:./\n");
        }
        $goods_id = intval($_REQUEST ['goods_id']);
        exit();
    }
    // 初始化返回数组
    $result = array(
        'error' => 0,
        'message' => '',
        'content' => '',
        'goods_id' => '',
        'product_spec' => ''
    );
    if (empty($all_goods)) {
        $result ['error'] = 1;
        die(json_encode($result));
    }

    $json_goods = stripslashes($all_goods);
    $json_goods = json_decode($json_goods);
//        $json = new EcsJson();
//        $all_goods = $json->decode($all_goods);

    try {
        // 先清空 结算车
        model('Order')->clear_cart();
        // 将每一个商品参数添加到 结算购物车
        foreach ($json_goods as $goods) {
            //对goods处理
            $result['goods_id'] = $goods->goods_id;
            $result['product_spec'] = $goods->spec;

            // 查询：系统启用了库存，检查输入的商品数量是否有效
            // 查询
            $arrGoods = $this->model->table('goods')->field('goods_name,goods_number,extension_code')->where('goods_id =' . $goods->goods_id)->find();
            $goodsnmber = model('Users')->get_goods_number($goods->goods_id);
            $goodsnmber+=$goods->number;
            if (intval(C('use_storage')) > 0 && $arrGoods ['extension_code'] != 'package_buy') {
                if ($arrGoods ['goods_number'] < $goodsnmber) {
                    $result['error'] = 1;
                    $result['message'] = sprintf(L('stock_insufficiency'), $arrGoods ['goods_name'], $arrGoods ['goods_number'], $arrGoods ['goods_number']);
                    if (C('use_how_oos') == 1){
                        $result['message'] =L('oos_tips');
                    }
                    die(json_encode($result));
                }
            }
            // 检查：商品数量是否合法
            if (!is_numeric($goods->number) || intval($goods->number) <= 0) {
                $result ['error'] = 1;
                $result ['message'] = L('invalid_number');
            } else {
                // 更新：添加到购物车    TODO： 添加uid信息
                if (model('Order')->addto_cart($goods->goods_id, $goods->number, $goods->spec, $goods->parent, $goods->sceneid, $goods->rfansid)) {
                    if (C('cart_confirm') > 2) {
                        $result ['message'] = '';
                    } else {
                        $result ['message'] = C('cart_confirm') == 1 ? L('addto_cart_success_1') : L('addto_cart_success_2');
                    }
                    $result ['content'] = insert_cart_info();
                    $result ['one_step_buy'] = C('one_step_buy');
                } else {
                    $result ['message'] = ECTouch::err()->last_message();
                    $result ['error'] = ECTouch::err()->error_no;
                    $result ['goods_id'] = stripslashes($goods->goods_id);
                    if (is_array($goods->spec)) {
                        $result ['product_spec'] = implode(',', $goods->spec);
                    } else {
                        $result ['product_spec'] = $goods->spec;
                    }
                }
            }
            $cart_confirm = C('cart_confirm');
            $result ['confirm_type'] = !empty($cart_confirm) ? C('cart_confirm') : 2;
            // 返回购物车商品总数量
            $result ['cart_number'] = insert_cart_info_number();
            if ($result ['error'] != 0) die(json_encode($result));

        }
        die(json_encode($result));
    }catch (Exception $e) {
        $result['error'] = '1';
        $result['message'] = '订单提交失败！ 请重试。';
        die(json_encode($result));
    }
}
```
ajax添加购物车返回没有错误，跳转index.php?m=default&c=flow&a=checkout，也就是订单确认页面。订单信息是根据结算车中的信息来生成的
```
public function checkout() {
//        $agent_id = $_SESSION['agent_id'];
        /* 取得购物类型 */
        $flow_type = isset($_SESSION ['flow_type']) ? intval($_SESSION ['flow_type']) : CART_GENERAL_GOODS;
        /* 团购标志 */
        if ($flow_type == CART_GROUP_BUY_GOODS) {
            $this->assign('is_group_buy', 1);
        } /* 积分兑换商品 */ elseif ($flow_type == CART_EXCHANGE_GOODS) {
            $this->assign('is_exchange_goods', 1);
        } else {
            // 正常购物流程 清空其他购物流程情况
            $_SESSION ['flow_order'] ['extension_code'] = '';
        }
        /* 检查购物车中是否有商品 */
//        $condition = "session_id = '" . SESS_ID . "' " . "AND parent_id = 0 AND is_gift = 0 AND rec_type = '$flow_type'";
        $condition = "user_id = '" . $_SESSION ['user_id'] . "' " . "AND parent_id = 0 AND is_gift = 0 AND rec_type = '$flow_type'";
        $count = $this->model->table('cart')->field('COUNT(*)')->where($condition)->getOne();
        if ($count == 0) {
            show_message(L('no_goods_in_cart'), '', '', 'warning');
        }

        //  检查用户是否已经登录 如果用户已经登录了则检查是否有默认的收货地址 如果没有登录则跳转到登录和注册页面
        if (empty($_SESSION ['direct_shopping']) && $_SESSION ['user_id'] == 0) {
            /* 用户没有登录且没有选定匿名购物，转向到登录页面 */
            //$this->redirect(url('user/login',array('step'=>'flow')));
            //自动登录qyl
            $this->beyond_redirect('login', 2);
            exit;
        }
        // 获取收货人信息
/*-------------------添加设置临时收货地址功能------Dennis--20170401-----------------*/
        if ($_GET['aid']) {
            $consignee = model('Users')->get_consignee_list($_SESSION ['user_id'], $_GET['aid']);
        }else {
            $consignee = model('Order')->get_consignee($_SESSION ['user_id']);
        }
/*-------------------END-----------------------*/
//        $consignee = model('Order')->get_consignee($_SESSION ['user_id']);

        /* 检查收货人信息是否完整 */
        if (!model('Order')->check_consignee_info($consignee, $flow_type)) {
            /* 如果不完整则转向到收货人信息填写界面 */
            ecs_header("Location: " . url('flow/consignee_list') . "\n");
        }
        // 获取配送地址
        $consignee_list = model('Users')->get_consignee_list($_SESSION ['user_id']);
        $this->assign('consignee_list', $consignee_list);
        //获取默认配送地址
        $address_id = $this->model->table('users')->field('address_id')->where("user_id = '" . $_SESSION['user_id'] . "' ")->getOne();
        $this->assign('address_id', $address_id);

        $_SESSION ['flow_consignee'] = $consignee;
        $this->assign('consignee', $consignee);

        /* 对商品信息赋值 */
        $cart_goods = model('Order')->cart_goods($flow_type); // 取得结算购物车商品列表，计算合计
        $this->assign('goods_list', $cart_goods);
        // 获取最近的门店ID，作为整个订单的门店ID
        $sceneid = -1;
        foreach ($cart_goods as $good) {
            if (!empty($good['sceneid'])) {
                $sceneid = $good['sceneid'];
            }
        }
        $this->assign('sceneid', $sceneid);

        /* 对是否允许修改购物车赋值 */
        if ($flow_type != CART_GENERAL_GOODS || C('one_step_buy') == '1') {
            $this->assign('allow_edit_cart', 0);
        } else {
            $this->assign('allow_edit_cart', 1);
        }

        // 取得购物流程设置
        $this->assign('config', C('CFG'));
        // 取得订单信息
        $order = model('Order')->flow_order_info();
        $this->assign('order', $order);

        /* 计算折扣 */
        if ($flow_type != CART_EXCHANGE_GOODS && $flow_type != CART_GROUP_BUY_GOODS) {
            $discount = model('Order')->compute_discount();
            $this->assign('discount', $discount ['discount']);
            $favour_name = empty($discount ['name']) ? '' : join(',', $discount ['name']);
            $this->assign('your_discount', sprintf(L('your_discount'), $favour_name, price_format($discount ['discount'])));
        }

        //计算订单的费用
        $total = model('Users')->order_fee($order, $cart_goods, $consignee);

        $this->assign('total', $total);
        $this->assign('shopping_money', sprintf(L('shopping_money'), $total ['formated_goods_price']));
        $this->assign('market_price_desc', sprintf(L('than_market_price'), $total ['formated_market_price'], $total ['formated_saving'], $total ['save_rate']));

        /* 取得可以得到的积分和红包 */
        $this->assign('total_integral', model('Order')->cart_amount(false, $flow_type) - $total ['bonus'] - $total ['integral_money']);
        $this->assign('total_bonus', price_format(model('Order')->get_total_bonus(), false));


        /* 取得配送列表 */
        $region = array(
            $consignee ['country'],
            $consignee ['province'],
            $consignee ['city'],
            $consignee ['district']
        );
        $shipping_list = model('Shipping')->available_shipping_list($region);
        $cart_weight_price = model('Order')->cart_weight_price($flow_type);
        $insure_disabled = true;
        $cod_disabled = true;

        // 查看购物车中是否全为免运费商品，若是则把运费赋为零
        $condition = "`session_id` = '" . SESS_ID . "' AND `extension_code` != 'package_buy' AND `is_shipping` = 0";
        $shipping_count = $this->model->table('cart')->field('count(*)')->where($condition)->getOne();
        foreach ($shipping_list as $key => $val) {

            $shipping_cfg = unserialize_config($val ['configure']);
            $shipping_fee = ($shipping_count == 0 and $cart_weight_price ['free_shipping'] == 1) ? 0 : shipping_fee($val['shipping_code'], unserialize($val ['configure']), $cart_weight_price ['weight'], $cart_weight_price ['amount'], $cart_weight_price ['number']);

            $shipping_list [$key] ['format_shipping_fee'] = price_format($shipping_fee, false);
            $shipping_list [$key] ['shipping_fee'] = $shipping_fee;
            $shipping_list [$key] ['free_money'] = price_format($shipping_cfg ['free_money'], false);
            $shipping_list [$key] ['insure_formated'] = strpos($val ['insure'], '%') === false ? price_format($val ['insure'], false) : $val ['insure'];

            /* 当前的配送方式是否支持保价 */
            if ($val ['shipping_id'] == $order ['shipping_id']) {
                $insure_disabled = ($val ['insure'] == 0);
                $cod_disabled = ($val ['support_cod'] == 0);
            }
	        // 兼容过滤ecjia配送方式
            if (substr($val['shipping_code'], 0 , 5) == 'ship_') {
                unset($shipping_list[$key]);
            }
        }

        $this->assign('shipping_list', $shipping_list);
        $this->assign('insure_disabled', $insure_disabled);
        $this->assign('cod_disabled', $cod_disabled);

        /* 取得支付列表 */
        if ($order ['shipping_id'] == 0) {
            $cod = true;
            $cod_fee = 0;
        } else {
            $shipping = model('Shipping')->shipping_info($order ['shipping_id']);
            $cod = $shipping ['support_cod'];

            if ($cod) {
                /* 如果是团购，且保证金大于0，不能使用货到付款 */
                if ($flow_type == CART_GROUP_BUY_GOODS) {
                    $group_buy_id = $_SESSION ['extension_id'];
                    if ($group_buy_id <= 0) {
                        show_message('error group_buy_id');
                    }
                    $group_buy = model('GroupBuyBase')->group_buy_info($group_buy_id);
                    if (empty($group_buy)) {
                        show_message('group buy not exists: ' . $group_buy_id);
                    }

                    if ($group_buy ['deposit'] > 0) {
                        $cod = false;
                        $cod_fee = 0;

                        /* 赋值保证金 */
                        $this->assign('gb_deposit', $group_buy ['deposit']);
                    }
                }

                if ($cod) {
                    $shipping_area_info = model('Shipping')->shipping_area_info($order ['shipping_id'], $region);
                    $cod_fee = $shipping_area_info ['pay_fee'];
                }
            } else {
                $cod_fee = 0;
            }
        }

        // 给货到付款的手续费加<span id>，以便改变配送的时候动态显示
        $payment_list = model('Order')->available_payment_list(1, $cod_fee);
        if (isset($payment_list)) {
            foreach ($payment_list as $key => $payment) {
                // 只保留显示手机版支付方式
                if(!file_exists(ROOT_PATH . 'plugins/payment/'.$payment['pay_code'].'.php')){
                    unset($payment_list[$key]);
                }
                if ($payment ['is_cod'] == '1') {
                    $payment_list [$key] ['format_pay_fee'] = '<span id="ECS_CODFEE">' . $payment ['format_pay_fee'] . '</span>';
                }

                /* 如果有易宝神州行支付 如果订单金额大于300 则不显示 */
                if ($payment ['pay_code'] == 'yeepayszx' && $total ['amount'] > 300) {
                    unset($payment_list [$key]);
                }
                /* 如果有余额支付 */
                if ($payment ['pay_code'] == 'balance') {
                    /* 如果未登录，不显示 */
                    if ($_SESSION ['user_id'] == 0) {
                        unset($payment_list [$key]);
                    } else {
                        if ($_SESSION ['flow_order'] ['pay_id'] == $payment ['pay_id']) {
                            $this->assign('disable_surplus', 1);
                        }
                    }
                }
                // 如果不是微信浏览器访问并且不是微信会员 则不显示微信支付
                /*if ($payment ['pay_code'] == 'wxpay' && !is_wechat_browser() && empty($_SESSION['openid'])) {
                    unset($payment_list [$key]);
                }*/
                // 兼容过滤ecjia支付方式
                if (substr($payment['pay_code'], 0 , 4) == 'pay_') {
                    unset($payment_list[$key]);
                }
            }
        }
        $this->assign('payment_list', $payment_list);

        /* 取得包装与贺卡 */
        if ($total ['real_goods_count'] > 0) {
            /* 只有有实体商品,才要判断包装和贺卡 */
            $use_package = C('use_package');
            if (!isset($use_package) || C('use_package') == '1') {
                /* 如果使用包装，取得包装列表及用户选择的包装 */
                $this->assign('pack_list', model('Order')->pack_list());
            }

            /* 如果使用贺卡，取得贺卡列表及用户选择的贺卡 */
            $use_card = C('use_card');
            if (!isset($use_card) || C('use_card') == '1') {
                $this->assign('card_list', model('Order')->card_list());
            }
        }

        $user_info = model('Order')->user_info($_SESSION ['user_id']);

        /* 如果使用余额，取得用户余额 */
        $use_surplus = C('use_surplus');
        if ((!isset($use_surplus) || C('use_surplus') == '1') && $_SESSION ['user_id'] > 0 && $user_info ['user_money'] > 0) {
            // 能使用余额
            $this->assign('allow_use_surplus', 1);
            $this->assign('your_surplus', $user_info ['user_money']);
        }

        /* 如果使用积分，取得用户可用积分及本订单最多可以使用的积分 */
        $use_integral = C('use_integral');
        if ((!isset($use_integral) || C('use_integral') == '1') && $_SESSION ['user_id'] > 0 && $user_info ['pay_points'] > 0 && ($flow_type != CART_GROUP_BUY_GOODS && $flow_type != CART_EXCHANGE_GOODS)) {
            // 能使用积分
            $this->assign('allow_use_integral', 1);
            $this->assign('order_max_integral', model('Flow')->flow_available_points()); // 可用积分
            $this->assign('your_integral', $user_info ['pay_points']); // 用户积分
        }

        /* 如果使用红包，取得用户可以使用的红包及用户选择的红包 */
        $use_bonus = C('use_bonus');
        if ((!isset($use_bonus) || C('use_bonus') == '1') && ($flow_type != CART_GROUP_BUY_GOODS && $flow_type != CART_EXCHANGE_GOODS)) {
            // 取得用户可用红包
            $user_bonus = model('Order')->user_bonus($_SESSION ['user_id'], $total ['goods_price']);
            if (!empty($user_bonus)) {
                foreach ($user_bonus as $key => $val) {
                    $user_bonus [$key] ['bonus_money_formated'] = price_format($val ['type_money'], false);
                }
                $this->assign('bonus_list', $user_bonus);
            }

            // 能使用红包
            $this->assign('allow_use_bonus', 1);
        }

        /* 如果使用缺货处理，取得缺货处理列表 */
        $use_how_oos = C('use_how_oos');
        if (!isset($use_how_oos) || $use_how_oos == '1') {
            $oos = L('oos');
            if (is_array($oos) && !empty($oos)) {
                $this->assign('how_oos_list', L('oos'));
            }
        }

        /* 如果能开发票，取得发票内容列表 */
        $can_invoice = C('can_invoice');
        $invoice_content = C('invoice_content');
        if ((!isset($can_invoice) || $can_invoice == '1') && isset($invoice_content) && trim($invoice_content) != '' && $flow_type != CART_EXCHANGE_GOODS) {
            $inv_content_list = explode("\n", str_replace("\r", '', C('invoice_content')));
            $this->assign('inv_content_list', $inv_content_list);
            $inv_type_list = array();
            $invoice_type = C('invoice_type');
            foreach ($invoice_type['type'] as $key => $type) {
                if (!empty($type)) {
                    $inv_type_list [$type] = $type . ' [' . floatval($invoice_type['rate'] [$key]) . '%]';
                }
            }
            $this->assign('inv_type_list', $inv_type_list);
        }

        // print_r($inv_type_list);
        $this->assign('order', $order);
        /* 保存 session */
        $_SESSION ['flow_order'] = $order;
        $this->assign('currency_format', C('currency_format'));
        $this->assign('integral_scale', C('integral_scale'));
        $this->assign('step', ACTION_NAME);
        model('Common')->assign_dynamic('shopping_flow');

        $this->assign('title', L('order_detail'));

        $this->display('flow.dwt');
    }
```
订单确认页面是一个form表单，点击立即购买跳转‘{:url('flow/done')}’，这才生成正式订单。这里会有个清空结算车的操作。在这个操作之前清空虚拟车。
```
public function done() {
        /* 取得购物类型 */
        $flow_type = isset($_SESSION ['flow_type']) ? intval($_SESSION ['flow_type']) : CART_GENERAL_GOODS;
        /* 检查购物车中是否有商品 */
//        $condition = " session_id = '" . SESS_ID . "' " . "AND parent_id = 0 AND is_gift = 0 AND rec_type = '$flow_type'";
        $condition = " user_id = '" . $_SESSION['user_id'] . "' " . "AND parent_id = 0 AND is_gift = 0 AND rec_type = '$flow_type'";
        $count = $this->model->table('cart')->field('COUNT(*)')->where($condition)->getOne();
        if ($count == 0) {
            show_message(L('no_goods_in_cart'), '', '', 'warning');
        }
        /* 如果使用库存，且下订单时减库存，则减少库存 */
        if (C('use_storage') == '1' && C('stock_dec_time') == SDT_PLACE) {
            $cart_goods_stock = model('Order')->get_cart_goods();
            $_cart_goods_stock = array();
            foreach ($cart_goods_stock ['goods_list'] as $value) {
                $_cart_goods_stock [$value ['rec_id']] = $value ['goods_number'];
            }
            model('Flow')->flow_cart_stock($_cart_goods_stock);
            unset($cart_goods_stock, $_cart_goods_stock);
        }
        // 检查用户是否已经登录 如果用户已经登录了则检查是否有默认的收货地址 如果没有登录则跳转到登录和注册页面
        if (empty($_SESSION ['direct_shopping']) && $_SESSION ['user_id'] == 0) {
            /* 用户没有登录且没有选定匿名购物，转向到登录页面 */
            ecs_header("Location: " . url('user/login') . "\n");
        }

        // 获取收货人信息
        $consignee = model('Order')->get_consignee($_SESSION ['user_id']);
        /* 检查收货人信息是否完整 */
        if (!model('Order')->check_consignee_info($consignee, $flow_type)) {
            /* 如果不完整则转向到收货人信息填写界面 */
            ecs_header("Location: " . url('flow/consignee') . "\n");
        }

        // 处理接收信息
        $how_oos = I('post.how_oos', 0);
        $card_message = I('post.card_message', trim, '');
        $inv_type = I('post.inv_type', '');
        $inv_payee = I('post.inv_payee', trim, '');
        $inv_content = I('post.inv_content', trim,'');
        $postscript = I('post.postscript',trim, '');
        $oos = L('oos.' . $how_oos);
        // 订单信息
        $order = array(
//          不支持选择配送方式固定为货到付'shipping_id' => I('post.shipping'), yql
            'shipping_id' => '7',
            'pay_id' => I('post.payment'), // 付款方式
            'pack_id' => I('post.pack', 0),
            'card_id' => isset($_POST ['card']) ? intval($_POST ['card']) : 0,
            'card_message' => $card_message,
            'surplus' => isset($_POST ['surplus']) ? floatval($_POST ['surplus']) : 0.00,
            'integral' => isset($_POST ['integral']) ? intval($_POST ['integral']) : 0,
            'bonus_id' => isset($_POST ['bonus']) ? intval($_POST ['bonus']) : 0,
            'need_inv' => empty($_POST ['need_inv']) ? 0 : 1,
            'inv_type' => $inv_type,
            'inv_payee' => $inv_payee,
            'inv_content' => $inv_content,
            'postscript' => $postscript,
            'how_oos' => isset($oos) ? addslashes("$oos") : '',
            'need_insure' => isset($_POST ['need_insure']) ? intval($_POST ['need_insure']) : 0,
            'user_id' => $_SESSION ['user_id'],
            'add_time' => gmtime(),
            'order_status' => OS_UNCONFIRMED,
            'shipping_status' => SS_UNSHIPPED,
            'pay_status' => PS_UNPAYED,
            'agency_id' => model('Order')->get_agency_by_regions(array(
                $consignee ['country'],
                $consignee ['province'],
                $consignee ['city'],
                $consignee ['district']
            )),
            'sceneid' => $_POST['sceneid'],
        );

        /* 扩展信息 */
        if (isset($_SESSION ['flow_type']) && intval($_SESSION ['flow_type']) != CART_GENERAL_GOODS) {
            $order ['extension_code'] = $_SESSION ['extension_code'];
            $order ['extension_id'] = $_SESSION ['extension_id'];
        } else {
            $order ['extension_code'] = '';
            $order ['extension_id'] = 0;
        }
        /* 检查积分余额是否合法 */
        $user_id = $_SESSION ['user_id'];
        if ($user_id > 0) {

            $user_info = model('Order')->user_info($user_id);
            $order ['surplus'] = min($order ['surplus'], $user_info ['user_money'] + $user_info ['credit_line']);
            if ($order ['surplus'] < 0) {
                $order ['surplus'] = 0;
            }

            // 查询用户有多少积分
            $flow_points = model('Flow')->flow_available_points(); // 该订单允许使用的积分
            $user_points = $user_info ['pay_points']; // 用户的积分总数

            $order ['integral'] = min($order ['integral'], $user_points, $flow_points);
            if ($order ['integral'] < 0) {
                $order ['integral'] = 0;
            }
        } else {
            $order ['surplus'] = 0;
            $order ['integral'] = 0;
        }

        /* 检查红包是否存在 */
        if ($order ['bonus_id'] > 0) {
            $bonus = model('Order')->bonus_info($order ['bonus_id']);
            if (empty($bonus) || $bonus ['user_id'] != $user_id || $bonus ['order_id'] > 0 || $bonus ['min_goods_amount'] > model('Order')->cart_amount(true, $flow_type)) {
                $order ['bonus_id'] = 0;
            }
        } elseif (isset($_POST ['bonus_sn'])) {
            $bonus_sn = trim($_POST ['bonus_sn']);
            $bonus = model('Order')->bonus_info(0, $bonus_sn);
            $now = gmtime();
            if (empty($bonus) || $bonus ['user_id'] > 0 || $bonus ['order_id'] > 0 || $bonus ['min_goods_amount'] > model('Order')->cart_amount(true, $flow_type) || $now > $bonus ['use_end_date']) {

            } else {
                if ($user_id > 0) {
                    $sql = "UPDATE " . $this->model->pre . "user_bonus SET user_id = '$user_id' WHERE bonus_id = '$bonus[bonus_id]' LIMIT 1";
                    $this->model->query($sql);
                }
                $order ['bonus_id'] = $bonus ['bonus_id'];
                $order ['bonus_sn'] = $bonus_sn;
            }
        }

        /* 订单中的商品 */
        $cart_goods = model('Order')->cart_goods($flow_type);
        if (empty($cart_goods)) {
            show_message(L('no_goods_in_cart'), L('back_home'), './', 'warning');
        }

        /* 检查商品总额是否达到最低限购金额 */
        if ($flow_type == CART_GENERAL_GOODS && model('Order')->cart_amount(true, CART_GENERAL_GOODS) < C('min_goods_amount')) {
            show_message(sprintf(L('goods_amount_not_enough'), price_format(C('min_goods_amount'), false)));
        }

        /* 收货人信息 */
        foreach ($consignee as $key => $value) {
            $order [$key] = addslashes($value);
        }


        /* 判断是不是实体商品 */
        foreach ($cart_goods as $val) {
            /* 统计实体商品的个数 */
            if ($val ['is_real']) {
                $is_real_good = 1;
            }
        }
        if (isset($is_real_good)) {
            $res = $this->model->table('shipping')->field('shipping_id')->where("shipping_id=" . $order ['shipping_id'] . " AND enabled =1")->getOne();
            if (!$res) {
                show_message(L('flow_no_shipping'));
            }
        }
        /* 订单中的总额 */
        $total = model('Users')->order_fee($order, $cart_goods, $consignee);
        $order ['bonus'] = $total ['bonus'];
        $order ['goods_amount'] = $total ['goods_price'];
        $order ['discount'] = $total ['discount'];
        $order ['surplus'] = $total ['surplus'];
        $order ['tax'] = $total ['tax'];

        // 购物车中的商品能享受红包支付的总额
        $discount_amout = model('Order')->compute_discount_amount();
        // 红包和积分最多能支付的金额为商品总额
        $temp_amout = $order ['goods_amount'] - $discount_amout;
        if ($temp_amout <= 0) {
            $order ['bonus_id'] = 0;
        }

        /* 配送方式 */
        if ($order ['shipping_id'] > 0) {
            $shipping = model('Shipping')->shipping_info($order ['shipping_id']);
            $order ['shipping_name'] = addslashes($shipping ['shipping_name']);
        }
        $order ['shipping_fee'] = $total ['shipping_fee'];
        $order ['insure_fee'] = $total ['shipping_insure'];

        /* 支付方式 */
        if ($order ['pay_id'] > 0) {
            $payment = model('Order')->payment_info($order ['pay_id']);
            $order ['pay_name'] = addslashes($payment ['pay_name']);
        }

        $order ['pay_fee'] = $total ['pay_fee'];
        $order ['cod_fee'] = $total ['cod_fee'];

        /* 商品包装 */
        if ($order ['pack_id'] > 0) {
            $pack = model('Order')->pack_info($order ['pack_id']);
            $order ['pack_name'] = addslashes($pack ['pack_name']);
        }
        $order ['pack_fee'] = $total ['pack_fee'];

        /* 祝福贺卡 */
        if ($order ['card_id'] > 0) {
            $card = model('Order')->card_info($order ['card_id']);
            $order ['card_name'] = addslashes($card ['card_name']);
        }
        $order ['card_fee'] = $total ['card_fee'];
        $order ['order_amount'] = number_format($total ['amount'], 2, '.', '');

        /* 如果全部使用余额支付，检查余额是否足够 */
        if ($payment ['pay_code'] == 'balance' && $order ['order_amount'] > 0) {
            if ($order ['surplus'] > 0) {    // 余额支付里如果输入了一个金额
                $order ['order_amount'] = $order ['order_amount'] + $order ['surplus'];
                $order ['surplus'] = 0;
            }
            if ($order ['order_amount'] > ($user_info ['user_money'] + $user_info ['credit_line'])) {
                show_message(L('balance_not_enough'));
            } else {
                $order ['surplus'] = $order ['order_amount'];
                $order ['order_amount'] = 0;
            }
        }

        /* 如果订单金额为0（使用余额或积分或红包支付），修改订单状态为已确认、已付款 */
        if ($order ['order_amount'] <= 0) {
            $order ['order_status'] = OS_CONFIRMED;
            $order ['confirm_time'] = gmtime();
            $order ['pay_status'] = PS_PAYED;
            $order ['pay_time'] = gmtime();
            $order ['order_amount'] = 0;
        }

        $order ['integral_money'] = $total ['integral_money'];
        $order ['integral'] = $total ['integral'];

        if ($order ['extension_code'] == 'exchange_goods') {
            $order ['integral_money'] = 0;
            $order ['integral'] = $total ['exchange_integral'];
        }

        $order ['from_ad'] = !empty($_SESSION ['from_ad']) ? $_SESSION ['from_ad'] : '0';
        $order ['referer'] = !empty($_SESSION ['referer']) ? addslashes($_SESSION ['referer']). 'Touch' : 'Touch';

        /* 记录扩展信息 */
        if ($flow_type != CART_GENERAL_GOODS) {
            $order ['extension_code'] = $_SESSION ['extension_code'];
            $order ['extension_id'] = $_SESSION ['extension_id'];
        }

        $parent_id = M()->table('users')->field('parent_id')->where("user_id=".$_SESSION['user_id'])->getOne();
        $order ['parent_id'] = $parent_id;

        /* 插入订单表 */
        $error_no = 0;
        do {
            $order ['order_sn'] = get_order_sn(); // 获取新订单号
            $new_order = model('Common')->filter_field('order_info', $order);
            $this->model->table('order_info')->data($new_order)->insert();
            $error_no = M()->errno();

            if ($error_no > 0 && $error_no != 1062) {
                die(M()->errorMsg());
            }
        } while ($error_no == 1062); // 如果是订单号重复则重新提交数据
        $new_order_id = M()->insert_id();
        $order ['order_id'] = $new_order_id;

        /* 插入订单商品 */
        $sql = "INSERT INTO {$this->model->pre}order_goods( order_id, goods_id, goods_name, goods_sn, product_id, goods_number, market_price, goods_price, goods_attr, is_real, extension_code, parent_id, is_gift, goods_attr_id, sceneid) " . " SELECT '$new_order_id', goods_id, goods_name, goods_sn, product_id, goods_number, market_price, goods_price, goods_attr, is_real, extension_code, parent_id, is_gift, goods_attr_id, sceneid FROM " . $this->model->pre . "cart WHERE session_id = '" . SESS_ID . "' AND rec_type = '$flow_type'";
        $this->model->query($sql);
        /* 修改拍卖活动状态 */
        if ($order ['extension_code'] == 'auction') {
            $sql = "UPDATE " . $this->model->pre . "goods_activity SET is_finished='2' WHERE act_id=" . $order ['extension_id'];
            $this->model->query($sql);
        }

        /* 处理余额、积分、红包 */
        if ($order ['user_id'] > 0 && $order ['surplus'] > 0) {
            model('ClipsBase')->log_account_change($order ['user_id'], $order ['surplus'] * (- 1), 0, 0, 0, sprintf(L('pay_order'), $order ['order_sn']));
        }
        if ($order ['user_id'] > 0 && $order ['integral'] > 0) {
            model('ClipsBase')->log_account_change($order ['user_id'], 0, 0, 0, $order ['integral'] * (- 1), sprintf(L('pay_order'), $order ['order_sn']));
        }

        if ($order ['bonus_id'] > 0 && $temp_amout > 0) {
            model('Order')->use_bonus($order ['bonus_id'], $new_order_id);
        }

        /* 如果使用库存，且下订单时减库存，则减少库存 */
        if (C('use_storage') == '1' && C('stock_dec_time') == SDT_PLACE) {
            model('Order')->change_order_goods_storage($order ['order_id'], true, SDT_PLACE);
        }

        /* 给商家发邮件 */
        /* 增加是否给客服发送邮件选项 */
        if (C('send_service_email') && C('service_email') != '') {
            $tpl = model('Base')->get_mail_template('remind_of_new_order');
            $this->assign('order', $order);
            $this->assign('goods_list', $cart_goods);
            $this->assign('shop_name', C('shop_name'));
            $this->assign('send_date', date(C('time_format')));
            $content = ECTouch::$view->fetch('str:' . $tpl ['template_content']);
            send_mail(C('shop_name'), C('service_email'), $tpl ['template_subject'], $content, $tpl ['is_html']);
        }

        /* 如果需要，发短信 */
        if (C('sms_order_placed') == '1' && C('sms_shop_mobile') != '') {
            $sms = new EcsSms();
            $msg = $order ['pay_status'] == PS_UNPAYED ? L('order_placed_sms') : L('order_placed_sms') . '[' . L('sms_paid') . ']';
            $sms->send(C('sms_shop_mobile'), sprintf($msg, $order ['consignee'], $order ['mobile']), '', 13, 1);
        }
         /* 如果需要，微信通知 by wanglu */
        /*if (method_exists('WechatController', 'snsapi_base') && is_wechat_browser()) {
            $order_url = __HOST__ . url('user/order_detail', array('order_id' => $order ['order_id']));
            $order_url = urlencode(base64_encode($order_url));
            send_wechat_message('order_remind', '', $order['order_sn'] . L('order_effective'), $order_url, $order['order_sn']);
        }*/
        /* 如果需要，微信通知 by wanglu */
        // if (method_exists('WechatController', 'do_oauth')) {
        //     $order_url = __HOST__ . url('user/order_detail', array('order_id' => $order ['order_id']));
        //     $order_url = urlencode(base64_encode($order_url));
        //     send_wechat_message('order_remind', '', $order['order_sn'] . L('order_effective'), $order_url, $order['order_sn']);
        // }
        /* 如果订单金额为0 处理虚拟卡 */
        if ($order ['order_amount'] <= 0) {
            $sql = "SELECT goods_id, goods_name, goods_number AS num FROM " . $this->model->pre . "cart WHERE is_real = 0 AND extension_code = 'virtual_card'" . " AND session_id = '" . SESS_ID . "' AND rec_type = '$flow_type'";
            $res = $this->model->query($sql);

            $virtual_goods = array();
            foreach ($res as $row) {
                $virtual_goods ['virtual_card'] [] = array(
                    'goods_id' => $row ['goods_id'],
                    'goods_name' => $row ['goods_name'],
                    'num' => $row ['num']
                );
            }

            if ($virtual_goods and $flow_type != CART_GROUP_BUY_GOODS) {
                /* 虚拟卡发货 */
                if (model('OrderBase')->virtual_goods_ship($virtual_goods, $msg, $order ['order_sn'], true)) {
                    /* 如果没有实体商品，修改发货状态，送积分和红包 */
                    $count = $this->model->table('order_goods')->field('COUNT(*)')->where("order_id = '$order[order_id]' " . " AND is_real = 1")->getOne();
                    if ($count <= 0) {
                        /* 修改订单状态 */
                        model('Users')->update_order($order ['order_id'], array(
                            'shipping_status' => SS_SHIPPED,
                            'shipping_time' => gmtime()
                        ));

                        /* 如果订单用户不为空，计算积分，并发给用户；发红包 */
                        if ($order ['user_id'] > 0) {
                            /* 取得用户信息 */
                            $user = model('Order')->user_info($order ['user_id']);

                            /* 计算并发放积分 */
                            $integral = model('Order')->integral_to_give($order);
                            model('ClipsBase')->log_account_change($order ['user_id'], 0, 0, intval($integral ['rank_points']), intval($integral ['custom_points']), sprintf(L('order_gift_integral'), $order ['order_sn']));

                            /* 发放红包 */
                            model('Order')->send_order_bonus($order ['order_id']);
                        }
                    }
                }
            }
        }

        // 销量
        model('Flow')->add_touch_goods($flow_type, $order ['extension_code']);

/*------------添加个人二维码分享折扣--20170526--Dennis------------------*/
        // 根据虚拟车中的refer_fansid更新 ecs_order_goods 的frefer
        model('Order')->update_goods_refer($new_order_id);

/*------------修改购物车逻辑--20170316--Dennis------------------*/
        // 在购物车清空之前，根据购物车里面的商品，删除 虚拟车 中同样商品的数据
        model('Order')->clear_virtual_cart();
/*------------END--20170316--Dennis------------------*/

        /* 清空购物车 */
        model('Order')->clear_cart($flow_type);


        /* 清除缓存，否则买了商品，但是前台页面读取缓存，商品数量不减少 */
        clear_all_files();

        /* 插入支付日志 */
        $order ['log_id'] = model('ClipsBase')->insert_pay_log($new_order_id, $order ['order_amount'], PAY_ORDER);

        /* 取得支付信息，生成支付代码 */
        if ($order ['order_amount'] > 0) {
            $payment = model('Order')->payment_info($order ['pay_id']);

            include_once (ROOT_PATH . 'plugins/payment/' . $payment ['pay_code'] . '.php');

            $pay_obj = new $payment ['pay_code'] ();

            $pay_online = $pay_obj->get_code($order, unserialize_config($payment ['pay_config']));

            $order ['pay_desc'] = $payment ['pay_desc'];

            $this->assign('pay_online', $pay_online);
        }
        if (!empty($order ['shipping_name'])) {
            $order ['shipping_name'] = trim(stripcslashes($order ['shipping_name']));
        }
        // 如果是银行汇款或货到付款 则显示支付描述
        if ($payment['pay_code'] == 'bank' || $payment['pay_code'] == 'cod'){
            if (empty($order ['pay_name'])) {
                $order ['pay_name'] = trim(stripcslashes($payment ['pay_name']));
            }
            $this->assign('pay_desc',$order['pay_desc']);
        }
        // 货到付款不显示
        if ($payment ['pay_code'] != 'balance') {
            /* 生成订单后，修改支付，配送方式 */

            // 支付方式
            $payment_list = model('Order')->available_payment_list(0);
            if (isset($payment_list)) {
                foreach ($payment_list as $key => $payment) {

                    /* 如果有易宝神州行支付 如果订单金额大于300 则不显示 */
                    if ($payment ['pay_code'] == 'yeepayszx' && $total ['amount'] > 300) {
                        unset($payment_list [$key]);
                    }
                    // 过滤掉当前的支付方式
                    if ($payment ['pay_id'] == $order ['pay_id']) {
                        unset($payment_list [$key]);
                    }
                    /* 如果有余额支付 */
                    if ($payment ['pay_code'] == 'balance') {
                        /* 如果未登录，不显示 */
                        if ($_SESSION ['user_id'] == 0) {
                            unset($payment_list [$key]);
                        } else {
                            if ($_SESSION ['flow_order'] ['pay_id'] == $payment ['pay_id']) {
                                $this->assign('disable_surplus', 1);
                            }
                        }
                    }
                    // 如果不是微信浏览器访问并且不是微信会员 则不显示微信支付
                    if ($payment ['pay_code'] == 'wxpay' && !is_wechat_browser() && empty($_SESSION['openid'])) {
                        unset($payment_list [$key]);
                    }
                    // 兼容过滤ecjia支付方式
                    if (substr($payment['pay_code'], 0 , 4) == 'pay_') {
                        unset($payment_list[$key]);
                    }
                }
            }
            $this->assign('payment_list', $payment_list);
            $this->assign('pay_code', 'no_balance');
        }

///*-----添加门店与订单关联信息----Dennis--20170327----*/
//        $agent_id = $_SESSION['agent_id'];
//        // 将当前订单 sn 和门店id 存入by_order_agent表
//        if (!empty($agent_id)) model('Order')->add_agent_order(intval($agent_id), $order['order_id'], $order['order_sn'], $order['goods_amount'], $order['order_amount']);
///*--END-------Dennis--20170327----*/

        /* 订单信息 */
        $this->assign('order', $order);

        $this->assign('total', $total);
        $this->assign('goods_list', $cart_goods);
        $this->assign('order_submit_back', sprintf(L('order_submit_back'), L('back_home'), L('goto_user_center'))); // 返回提示

        user_uc_call('add_feed', array($order ['order_id'], BUY_GOODS)); // 推送feed到uc
        unset($_SESSION ['flow_consignee']); // 清除session中保存的收货人信息
        unset($_SESSION ['flow_order']);
        unset($_SESSION ['direct_shopping']);

        $this->assign('currency_format', C('currency_format'));
        $this->assign('integral_scale', C('integral_scale'));
        $this->assign('step', ACTION_NAME);

        $this->assign('title', L('order_submit'));
        $this->display('flow.dwt');
    }
```
到此，订单生成，该修改完成了。  
### 关键点有两个：
1. 新建虚拟车表，记录购物车中的数据，结算车始终保持空的
2. 根据勾选的虚拟车商品插入结算车前，清空结算车；订单生成后，根据结算车清空虚拟车

应该有比这个更简单的方案。  
例如直接用购物车勾选的内容来生成订单，不走之前生成订单的逻辑。  
本人对ectouch的修改把握不大，就用了比较繁琐保守的修改方法。