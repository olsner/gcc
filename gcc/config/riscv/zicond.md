;; Machine description for the RISC-V Zicond extension and functionally-
;; equivalent XVentanaCondOps vendor extension
;; Copyright (C) 2022-23 Free Software Foundation, Inc.

;; This file is part of GCC.

;; GCC is free software; you can redistribute it and/or modify
;; it under the terms of the GNU General Public License as published by
;; the Free Software Foundation; either version 3, or (at your option)
;; any later version.

;; GCC is distributed in the hope that it will be useful,
;; but WITHOUT ANY WARRANTY; without even the implied warranty of
;; MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
;; GNU General Public License for more details.

;; You should have received a copy of the GNU General Public License
;; along with GCC; see the file COPYING3.  If not see
;; <http://www.gnu.org/licenses/>.

(define_code_iterator eq_or_ne [eq ne])
(define_code_attr eqz [(eq "nez") (ne "eqz")])
(define_code_attr nez [(eq "eqz") (ne "nez")])
(define_code_attr eqz_ventana [(eq "n") (ne "")])
(define_code_attr nez_ventana [(eq "") (ne "n")])

;; Zicond / XVentanaCondOps
(define_insn "*czero.<eqz>.<GPR:mode><X:mode>"
  [(set (match_operand:GPR 0 "register_operand"                      "=r")
        (if_then_else:GPR (eq_or_ne (match_operand:X 1 "register_operand" "r")
                                    (const_int 0))
                          (match_operand:GPR 2 "register_operand"    "r")
                          (const_int 0)))]
  "TARGET_ZICOND_LIKE"
  {
    if (TARGET_ZICOND)
      return "czero.<eqz>\t%0,%2,%1";
    else if (TARGET_XVENTANACONDOPS && TARGET_64BIT)
      return "vt.maskc<eqz_ventana>\t%0,%2,%1";
    else
      gcc_unreachable ();
  }
)

(define_insn "*czero.<nez>.<GPR:mode><X:mode>"
  [(set (match_operand:GPR 0 "register_operand"                     "=r")
        (if_then_else:GPR (eq_or_ne (match_operand:X 1 "register_operand" "r")
                                    (const_int 0))
                          (const_int 0)
                          (match_operand:GPR 2 "register_operand"   "r")))]
  "TARGET_ZICOND_LIKE"
  {
    if (TARGET_ZICOND)
      return "czero.<nez>\t%0,%2,%1";
    else if (TARGET_XVENTANACONDOPS && TARGET_64BIT)
      return "vt.maskc<nez_ventana>\t%0,%2,%1";
    else
      gcc_unreachable ();
  }
)

;; Special optimization under eq/ne in primitive semantics
(define_insn "*czero.eqz.<GPR:mode><X:mode>.opt1"
  [(set (match_operand:GPR 0 "register_operand"                   "=r")
        (if_then_else:GPR (eq (match_operand:X 1 "register_operand" "r")
                              (const_int 0))
                          (match_operand:GPR 2 "register_operand" "1")
                          (match_operand:GPR 3 "register_operand" "r")))]
  "TARGET_ZICOND_LIKE && rtx_equal_p (operands[1], operands[2])"
  {
    if (TARGET_ZICOND)
      return "czero.eqz\t%0,%3,%1";
    else if (TARGET_XVENTANACONDOPS && TARGET_64BIT)
      return "vt.maskc\t%0,%3,%1";
    else
      gcc_unreachable ();
  }
)

(define_insn "*czero.nez.<GPR:mode><X:mode>.opt2"
  [(set (match_operand:GPR 0 "register_operand"                   "=r")
        (if_then_else:GPR (ne (match_operand:X 1 "register_operand" "r")
                              (const_int 0))
                          (match_operand:GPR 2 "register_operand" "r")
                          (match_operand:GPR 3 "register_operand" "1")))]
  "TARGET_ZICOND && rtx_equal_p (operands[1], operands[3])"
  {
    if (TARGET_ZICOND)
      return "czero.eqz\t%0,%2,%1";
    else if (TARGET_XVENTANACONDOPS && TARGET_64BIT)
      return "vt.maskc\t%0,%2,%1";
    else
      gcc_unreachable ();
  }
)

;; Combine creates this form in some cases (particularly the coremark
;; CRC loop).
(define_split
  [(set (match_operand:X 0 "register_operand")
	(and:X (sign_extract:X (match_operand:X 1 "register_operand")
			       (const_int 1)
			       (match_operand 2 "immediate_operand"))
	       (match_operand:X 3 "register_operand")))
   (clobber (match_operand:X 4 "register_operand"))]
  "TARGET_ZICOND_LIKE && TARGET_ZBS"
  [(set (match_dup 4) (zero_extract:X (match_dup 1) (const_int 1) (match_dup 2)))
   (set (match_dup 0) (if_then_else:X (eq:X (match_dup 4) (const_int 0))
				      (const_int 0)
				      (match_dup 3)))])

(define_split
  [(set (match_operand:X 0 "register_operand")
	(and:X (sign_extract:X (match_operand:X 1 "register_operand")
			       (const_int 1)
			       (match_operand 2 "immediate_operand"))
	       (match_operand:X 3 "register_operand")))
   (clobber (match_operand:X 4 "register_operand"))]
  "TARGET_ZICOND_LIKE && !TARGET_ZBS && (UINTVAL (operands[2]) < 11)"
  [(set (match_dup 4) (and:X (match_dup 1) (match_dup 2)))
   (set (match_dup 0) (if_then_else:X (eq:X (match_dup 4) (const_int 0))
				      (const_int 0)
				      (match_dup 3)))]
{
  operands[2] = GEN_INT (1 << UINTVAL(operands[2]));
})
