https://gitee.com/src-openeuler/boost/issues/I466OU?from=project-issue

模式串：(?xi)|?+#

字符串：*



1.boost

boost-1.73 assert failed

boost-1.77 不匹配（错误，应该抛出异常）

2.pcre2

[root@45ea9e56a3b4 try]# ./a.out "(?xi)|?+#" "*"
PCRE2 compilation failed at offset 6: quantifier does not follow a repeatable item

3.grep -P

[root@45ea9e56a3b4 try]# grep -P "(?xi)|?+#" text 
grep: nothing to repeat



目前结论：

boost-1.77 不会报assert错误，但是不匹配的结论是错误的，应该在匹配模式串时就抛出异常



尝试：

(?xi) 匹配空串

(?i) 匹配空串

(?x)|?+# 抛异常

(?i)|?+# 不匹配（理论上应该抛出异常）

(?x:i)|?+# 抛异常

(?:)|?+ 抛异常





x表示模式中#后面的空格和文本将被忽略

i表示不区分大小写

查看匹配模式串的算法源码



boost::regex e(regex_string); //这里应该就抛出异常

namespace boost{

\#ifdef BOOST_REGEX_NO_FWD

typedef basic_regex<char, regex_traits<char> > regex;

\#ifndef BOOST_NO_WREGEX

typedef basic_regex<wchar_t, regex_traits<wchar_t> > wregex;

\#endif

\#endif



boost::regex -> 

basic_regex构造函数-> 

this->compile_ ->

detail::static_compile -> 

static_compile_impl1（default/specified traits）->

static_compile_impl2

```c
template<typename Xpr, typename BidiIter, typename Traits>
void static_compile_impl2(Xpr const &xpr, shared_ptr<regex_impl<BidiIter> > const &impl, Traits const &tr)
{
    typedef typename iterator_value<BidiIter>::type char_type;
    impl->tracking_clear();
    impl->traits_ = new traits_holder<Traits>(tr);

    // "compile" the regex and wrap it in an xpression_adaptor.
    typedef xpression_visitor<BidiIter, mpl::false_, Traits> visitor_type;
    visitor_type visitor(tr, impl);
    intrusive_ptr<matchable_ex<BidiIter> const> adxpr = make_adaptor<matchable_ex<BidiIter> >(
        typename Grammar<char_type>::template impl<Xpr const &, end_xpression, visitor_type &>()(
            xpr
            , end_xpression()
            , visitor
        )
    );

    // Link and optimize the regex
    common_compile(adxpr, *impl, visitor.traits());

    // References changed, update dependencies.
    impl->tracking_update();
}
```



parse_repeat  

-> switch(this->m_last_state->type)  

boost::re_detail_500::syntax_element_toggle_case

-> do nothing，break



(?i)|?+#

the case has changed in one or more of the alternatives within the scoped (...) block: we have to add a state to reset the case sensitivity

在作用域（…）块内的一个或多个备选方案中，案例发生了变化：我们必须添加一个状态来重置案例敏感度

the start of this alternative must have a case changes state if the current block has messed around with case changes

如果当前块的大小写更改混乱，则此备选方案的开头必须具有大小写更改状态



parse_all (this->*m_parser_proc)()

-> parse_extended 

-> case regex_constants::syntax_open_mark: return parse_open_paren();  // (?xi)|?+#

-> case regex_constants::syntax_close_mark: return false; //)|?+#

回到parse_all，return false

-> parse_alt // 解析| 得到?+#

-> parse_extend

-> case regex_constants::syntax_or:  return parse_alt() // ?+#

-> case regex_constants::syntax_question: 

m_position：?+# 

m_base:  (?xi)|?+#

++m_position; // +#

parse_repeat(0,1)



(?i)|?+# 时 m_has_case_change=true

opts & regbase::icase = 1048576

opts 1048576

regbase::icase 1048576

(?x)|?+#时 m_has_case_change=false

opts & regbase::icase = 0

opts 2048 

regbase::icase 1048576



```c
if(m_has_case_change)
{
    static_cast<re_case*>(
        this->append_state(syntax_element_toggle_case, sizeof(re_case))
    )->icase = this->m_icase;
}
```

m_has_case_change为true时会增加syntax_element_toggle_case的状态；

m_has_case_change为false时会增加syntax_element_toggle_case的状态



parse_alt:

the start of this alternative must have a case changes state if the current block has messed around with case changes:

如果当前块的大小写更改混乱，则此备选方案的开头必须具有大小写更改状态

-----------------------------------------------------------

理解为：“|”后的表达式要沿用具备不区分大小写的状态，所以要增加syntax_element_toggle_case

但是如果“|”后的表达式是以“?”开头就报错误



Add judgment of sub-expression

Determines whether the first character of the subexpression is a question mark



看syntax_element_toggle_case流程，状态转移有问题，

(a)|?+#会报错