---
layout: model
title: Rat growth
model-status: code
model-category: Undirected Constraints
model-tags: 
---

    (define ratsdata 
      '((151 199 246 283 320) (145 199 249 293 354)
        (147 214 263 312 328) (155 200 237 272 297)
        (135 188 230 280 323) (159 210 252 298 331)
        (141 189 231 275 305) (159 201 248 297 338)
        (177 236 285 350 376) (134 182 220 260 296)
        (160 208 261 313 352) (143 188 220 273 314)
        (154 200 244 289 325) (171 221 270 326 358)
        (163 216 242 281 312) (160 207 248 288 324)
        (142 187 234 280 316) (156 203 243 283 317)
        (157 212 259 307 336) (152 203 246 286 321)
        (154 205 253 298 334) (139 190 225 267 302)
        (146 191 229 272 302) (157 211 250 285 323)
        (132 185 237 286 331) (160 207 257 303 345)
        (169 216 261 295 333) (157 205 248 289 316)
        (137 180 219 258 291) (153 200 244 286 324)))
    
    [define x '(8.0 15.0 22.0 29.0 36.0)]
    [define xbar 22.0]
    [define gauss-factor (make-factor (lambda (m v x) (gauss-log-pdf m v x)))]
    
    (mh-query
     10 10000
     [define alphac (gaussian 0 10000)]
     [define betac (gaussian 0 10000)]
     [define tauc (abs (gaussian 0 100))]
     [define taualpha (abs (gaussian 0 100))]
     [define taubeta (abs (gaussian 0 100))]
     (- alphac (* betac xbar))
     [begin 
       (map (lambda (s) 
              (letrec(
                      [alpha-rat (gaussian alphac (/ 1.0 taualpha))]
                      [beta-rat (gaussian betac (/ 1.0 taubeta))]
                      [y-constrs
                       (map (lambda (x-w)
                              (gauss-factor (+ alpha-rat (* beta-rat (- (car x-w) xbar))) tauc (cadr x-w)))
                            (zip x s))])
                1.0
                ))
            ratsdata)
       true]
     )
 
Source: [shred](https://github.com/LFY/shred/blob/master/tests/rats.church)
